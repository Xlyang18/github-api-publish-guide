# 从受限网络发布项目到 GitHub（纯 API 方案）

> **适用场景**：所在网络访问 `github.com` 主站超时/被墙，但 `api.github.com` 仍可用。
> 典型表现：`git push` 卡住或 `unexpected EOF`；`gh auth login --web` 报 `Post "https://github.com/login/device/code": unexpected EOF`；但 `curl https://api.github.com` 正常返回。

**结论**：全程绕开 `github.com`，只用 REST API 完成「建仓库 → 推代码 → 打 tag → 发 Release → 传附件 → 校验」。

---

## 适用性自检

先跑这三条，确认你的网络属于哪种情况：

```bash
for h in api.github.com github.com uploads.github.com; do
  printf "%-22s " "$h"
  curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" --max-time 8 "https://$h"
done
```

| 结果 | 含义 | 能否用本方案 |
|---|---|---|
| `api` 200、`github` 000、`uploads` 302 | 典型受限网络 | ✅ 完全适用 |
| 三个都 200 | 网络正常 | 直接用 `git push` 即可，本方案仅作备用 |
| `api` 也 000 | 全站不可达 | ❌ 需要代理，本方案无效 |

---

## 前置：只需要一个 PAT

在 https://github.com/settings/tokens/new 生成 **classic token**，勾选：

- ✅ `repo` —— 建仓库、提交、发 Release
- ✅ `workflow` —— 仓库含 `.github/workflows/` 时必需，否则推送该文件会被拒

> `gh auth login --with-token` 会额外要求 `read:org` 权限，所以**不要用 gh CLI**，直接 curl 更省事。

存好并只让本机可读：

```bash
mkdir -p ~/.config/github
printf 'ghp_xxxxxxxx' > ~/.config/github/token
chmod 600 ~/.config/github/token
export TOKEN=$(head -1 ~/.config/github/token)
export OWNER=你的用户名
export REPO=你的仓库名      # 必须是 ASCII，见坑 1
```

---

## 八步流程

### 1. 验证 token

```bash
curl -sS -I -H "Authorization: Bearer $TOKEN" https://api.github.com/user \
  | grep -iE '^(HTTP/|x-oauth-scopes:)'
```

期望：`HTTP/2 200` + `x-oauth-scopes: repo, workflow`。

### 2. 建仓库

```bash
curl -sS -X POST https://api.github.com/user/repos \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"myproject","description":"一句话描述","private":false,
       "has_issues":true,"has_wiki":false,"has_projects":false,"auto_init":false}'
```

### 3. 造一个初始提交（必做）

**空仓库无法创建 blob**，`POST /git/blobs` 会返回 `409 Git Repository is empty`。

```bash
curl -sS -X PUT "https://api.github.com/repos/$OWNER/$REPO/contents/.init" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"message":"chore: 初始化仓库","content":"aW5pdAo="}'
```

推完正文后删掉它：

```bash
SHA=$(curl -sS -H "Authorization: Bearer $TOKEN" \
      "https://api.github.com/repos/$OWNER/$REPO/contents/.init" \
      | python3 -c "import sys,json;print(json.load(sys.stdin)['sha'])")
curl -sS -X DELETE "https://api.github.com/repos/$OWNER/$REPO/contents/.init" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"message\":\"chore: 移除占位\",\"sha\":\"$SHA\",\"branch\":\"main\"}"
```

### 4. 逐文件推送

**不要用 Git Data API 一次批量推**——在受限网络下极易 socket timeout，且失败要全部重来。
改用 **Contents API 逐文件 PUT**，天然断点续传。

```bash
STATE=/tmp/push-$REPO.done
while IFS= read -r f; do
  [ "$f" = ".init" ] && continue
  grep -qxF "$f" "$STATE" 2>/dev/null && continue

  B64=$(base64 < "$f" | tr -d '\n')
  Q=$(python3 -c "import sys,urllib.parse;print(urllib.parse.quote(sys.argv[1]))" "$f")

  # 文件已存在时 API 要求带上 sha，否则 422
  SHA=$(curl -sS -H "Authorization: Bearer $TOKEN" \
        "https://api.github.com/repos/$OWNER/$REPO/contents/$Q?ref=main" \
        | python3 -c "import sys,json;print(json.load(sys.stdin).get('sha',''))" 2>/dev/null)

  if [ -n "$SHA" ]; then
    BODY=$(python3 -c "import json,sys;print(json.dumps({'message':'chore: 更新 '+sys.argv[1],'content':sys.argv[2],'branch':'main','sha':sys.argv[3]}))" "$f" "$B64" "$SHA")
  else
    BODY=$(python3 -c "import json,sys;print(json.dumps({'message':'chore: 添加 '+sys.argv[1],'content':sys.argv[2],'branch':'main'}))" "$f" "$B64")
  fi

  curl -sS -X PUT "https://api.github.com/repos/$OWNER/$REPO/contents/$Q" \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "$BODY" > /dev/null
  echo "$f" >> "$STATE"
  echo "✓ $f"
done < <(git ls-files)
```

**实测速度**：约 100 个文件 → 5–10 分钟。放后台跑，别占前台。

### 5. 校验文件数

```bash
REMOTE=$(curl -sS -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/git/trees/main?recursive=1" \
  | python3 -c "import sys,json;print(len([x for x in json.load(sys.stdin)['tree'] if x['type']=='blob']))")
LOCAL=$(git ls-files | grep -v '^\.init$' | wc -l | tr -d ' ')
echo "远端 $REMOTE / 本地 $LOCAL"
[ "$REMOTE" = "$LOCAL" ] && echo "✅ 一致" || echo "⚠️ 不一致，回第 4 步补推"
```

### 6. 打 tag

**必须传完整 40 位 SHA**，短 SHA 会报 `At least 40 characters are required`。

```bash
SHA=$(curl -sS -H "Authorization: Bearer $TOKEN" \
      "https://api.github.com/repos/$OWNER/$REPO/git/refs/heads/main" \
      | python3 -c "import sys,json;print(json.load(sys.stdin)['object']['sha'])")

# 幂等保护：已存在就跳过（重复 POST 会 422）
if [ "$(curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN" \
      "https://api.github.com/repos/$OWNER/$REPO/git/refs/tags/v1.0.0")" != "200" ]; then

  TAGJ=$(python3 -c "import json,sys;print(json.dumps({'tag':sys.argv[1],'message':sys.argv[1],'object':sys.argv[2],'type':'commit'}))" "v1.0.0" "$SHA")
  TSHA=$(curl -sS -X POST "https://api.github.com/repos/$OWNER/$REPO/git/tags" \
         -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "$TAGJ" \
         | python3 -c "import sys,json;print(json.load(sys.stdin)['sha'])")

  curl -sS -X POST "https://api.github.com/repos/$OWNER/$REPO/git/refs" \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d "{\"ref\":\"refs/tags/v1.0.0\",\"sha\":\"$TSHA\"}"
fi
```

### 7. 发 Release + 传附件

```bash
# 幂等保护：已存在就复用
RESP=$(curl -sS -H "Authorization: Bearer $TOKEN" \
       "https://api.github.com/repos/$OWNER/$REPO/releases/tags/v1.0.0")
if echo "$RESP" | grep -q '"message":"Not Found"'; then
  RESP=$(curl -sS -X POST "https://api.github.com/repos/$OWNER/$REPO/releases" \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d "$(python3 -c "import json,sys;print(json.dumps({'tag_name':'v1.0.0','name':'v1.0.0','body':open('RELEASE_NOTES.md').read(),'draft':False,'prerelease':False}))")")
fi

UP=$(echo "$RESP" | python3 -c "import sys,json;print(json.load(sys.stdin)['upload_url'].split('{')[0])")

# 传附件：文件名必须是 ASCII，100MB 约 5–8 分钟
curl -sS --max-time 900 -X POST "$UP?name=myapp-1.0.0-arm64.dmg" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/octet-stream" \
  --data-binary "@./build/myapp-1.0.0-arm64.dmg"
```

### 8. 校验 Release

```bash
curl -sS -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/releases/tags/v1.0.0" \
  | python3 -c "
import sys,json
d=json.load(sys.stdin)
print(d['html_url'])
for a in d['assets']:
    print(f\"  {a['name']}  {a['size']/1048576:.0f} MB  {a['state']}\")"
```

必须全部 `state=uploaded`，且 `size` 与本地 `ls -l` 字节数一致。

---

## 坑位清单

| # | 现象 | 原因 | 对策 |
|---|---|---|---|
| 1 | 仓库名变成 `-` | GitHub 会过滤非 ASCII 字符，且不报错 | **仓库名只用 ASCII**，中文放 description / README |
| 2 | 附件名变成 `-1.0.0-arm64.dmg` | 同上 | **附件先重命名成 ASCII** 再上传 |
| 3 | `409 Git Repository is empty` | 空仓库无 commit，不能建 blob | 先做第 3 步初始化提交 |
| 4 | socket timeout / 传一半断掉 | Git Data API 批量推在受限网络不稳 | 逐文件 Contents API + `$STATE` 断点续传 |
| 5 | README 里图片全 404 | `.gitignore` 的 `screenshots/` 会匹配**任意层级**，把 `docs/screenshots/` 也忽略了 | 写 `/screenshots/`（带前导斜杠只匹配根目录） |
| 6 | `UnicodeEncodeError: 'ascii' codec` | Python 3.9 的 `urlopen` 不接受非 ASCII URL | 路径用 `urllib.parse.quote()` 编码 |
| 7 | `At least 40 characters are required` | tag ref 传了短 SHA | 用完整 40 位 SHA |
| 8 | `422 Reference already exists` | tag 不幂等 | 重跑前先 `GET /git/refs/tags/*` 判断 |
| 9 | `422 Release already_exists` | release 不幂等 | 重跑前先 `GET /releases/tags/*`，或先删同名资产 |
| 10 | 单文件 `413` / 超时 | Contents API 单文件建议 <50MB | 大文件只走 Release 附件，不放仓库 |

---

## 验证下载链接时别被骗

`https://github.com/.../releases/download/...` 在受限网络下 **curl 必然超时（`000`），但不代表资产坏了**。

**以 API 返回为准**：

```bash
# state 必须是 uploaded，size 必须与本地一致
curl -sS -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/releases/tags/v1.0.0" \
  | python3 -c "import sys,json;[print(a['name'],a['size'],a['state']) for a in json.load(sys.stdin)['assets']]"
```

链接对正常网络的访问者是有效的。

---

## 常见返工

| 场景 | 做法 |
|---|---|
| 附件名错了 | `DELETE /releases/assets/<id>` 删旧 → 本地改 ASCII 名 → 重传 |
| 文件漏推 | 修 `.gitignore` → `git add` → 重跑第 4 步（自动跳过已推）→ 第 5 步校验 |
| 文档需更新 | 改本地文件 → 重跑第 4 步 |
| 仓库名建错 | `PATCH /repos/$OWNER/$REPO -d '{"name":"newname"}'` |

---

## 为什么不用这些方案

| 方案 | 为什么不行 |
|---|---|
| `git push https://...` | 需要访问 `github.com`，主站不通 |
| `gh auth login --web` | device code 请求打到 `github.com/login/device/code`，直接 EOF |
| `gh auth login --with-token` | 额外要求 `read:org` 权限；有 token 还不如直接 curl |
| Git Data API 批量推 | 一次推上百文件，单次请求超时即全废 |
| SSH（`git@github.com`） | 需要 `github.com:22` 或 `ssh.github.com:443`，未必通；本方案不依赖它 |

---

## 许可

MIT

---

## 补充：又踩了 4 个（写脚本时才会遇到）

| # | 现象 | 原因 | 对策 |
|---|---|---|---|
| 11 | `Argument list too long`，只有大文件（png/icns）失败 | base64 后超 `ARG_MAX`（约 1MB），不能当命令行参数传 | base64 写**临时文件**，python 用 `open(os.environ['B64FILE'])` 读；curl 用 `--data-binary @文件` |
| 12 | `ok： unbound variable`，推到一半才崩 | `echo "$ok，..."` 中文紧跟变量名，被当成 `$ok，` | 中文前一律用 `${var}` |
| 13 | Python `SyntaxError`，但 `bash -n` 检查通过 | `"$(python3 -c "...{'k':v}...")"` 里花括号被 shell 当展开吃掉，**只在运行时炸** | JSON 生成器写成**独立 .py 脚本**，shell 只传 `--key value`，别内联字典 |
| 14 | tag 创建成功但 `object` 为空 | `/git/refs/heads/main` 的 sha 在 **`object.sha`**，不在顶层 | 用点路径取值，别用 `d.get('sha')` |

### 第 10~14 条的共同教训

前 10 个坑是**语义错**，这 4 个是**工程错**——`bash -n` 查不出来，只有真跑才暴露。

所以：**任何「上传脚本」都必须对真实仓库跑一遍完整流程**，包括大文件和中途崩溃的场景。
