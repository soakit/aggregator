# 所有对话和文档都使用中文

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a proxy aggregator tool ("代理池") that automatically crawls, collects, and validates proxy subscriptions from various sources. It registers temporary accounts on free airport services, collects their subscription links, validates proxies, and publishes them to GitHub Gist.

**Primary Language**: Python 3.x

**Key Purpose**: Educational crawler project that collects free proxy nodes from temporary airport services, validates them using Clash/Mihomo, and converts to multiple proxy client formats.

## Core Architecture

### Main Entry Points

**`subscribe/collect.py`** - 主执行脚本，带命令行接口 (subscribe:collect.py:1)

- 通过 `aggregate()` 函数编排整个收集 → 验证 → 转换 → 上传流程 (subscribe:collect.py:198)
- 命令：`python3 -u subscribe/collect.py [options]`
- 适用于：简单的订阅收集和聚合场景

**`subscribe/process.py`** - 高级配置模式的处理脚本

- 通过 `ProcessConfig` 类加载 JSON/YAML 配置文件 (subscribe:process.py:38)
- 支持更复杂的配置：多组订阅、爬虫配置、持久化存储等
- 命令：`python3 -u subscribe/process.py [options]`
- 适用于：需要精细控制的高级聚合场景

### Data Flow Pipeline

```
1. Collection (crawl.py, scripts/*)
   ↓
2. Registration (airport.py)
   ↓
3. Subscription Parsing (workflow.py)
   ↓
4. Proxy Validation (clash.py via Clash binary)
   ↓
5. Format Conversion (subconverter.py via subconverter binary)
   ↓
6. Publishing (push.py to GitHub Gist)
```

### Key Modules

**subscribe/crawl.py**

- Multi-threaded crawler for discovering subscriptions from multiple sources (subscribe:crawl.py:105)
- `batch_crawl()` orchestrates crawling from Google, Yandex, Telegram, Twitter, GitHub, and custom pages (subscribe:crawl.py:105)
- `multi_thread_crawl()` helper aggregates results from parallel plugin execution (subscribe:crawl.py:72)
- Plugin system in `subscribe/scripts/*.py` - custom scrapers implementing specific crawling strategies
  - Plugins invoked via `batch_call()` using `execute_script()` (subscribe:crawl.py:1762)
  - Each plugin should return list of dicts with subscription URLs and metadata
- Uses `SINGLE_LINK_FLAG = "singlelink://"` to aggregate individual proxy links (subscribe:crawl.py:60)
- `extract_subscribes()` parses subscription links from HTML/text content (subscribe:crawl.py:1069)
- `validate()` checks subscription availability and expiration (subscribe:crawl.py:1174)

**subscribe/airport.py**

- Core `AirPort` class handles airport registration and subscription extraction (subscribe:airport.py:129)
- `get_subscribe()` automated account registration with temporary email via `mailtm.py` (subscribe:airport.py:363)
- Handles email domain whitelists, coupons, and invite codes
- `RegisterRequire` dataclass tracks registration constraints (verify, invite, recaptcha, whitelist) (subscribe:airport.py:88)
- `parse()` parses various subscription formats (base64, clash YAML, etc.) (subscribe:airport.py:438)
- `decode()` static method converts subscription text to proxy list using subconverter (subscribe:airport.py:626)

**subscribe/clash.py**

- `generate_config()` creates Clash config files for proxy validation (subscribe:clash.py:44)
- `check()` function tests each proxy's latency and availability via Clash API (subscribe:clash.py:655)
- `filter_proxies()` deduplicates based on server:port combinations (subscribe:clash.py:65)
- `EXTERNAL_CONTROLLER = "127.0.0.1:9090"` - Clash API endpoint (subscribe:clash.py:33)
- `QuotedStr` class + `quoted_scalar()` handle special YAML string quoting to avoid Mihomo REALITY ID errors (subscribe:clash.py:36)
- `verify()` validates proxy dict structure and protocol-specific fields (subscribe:clash.py:293)
- `is_mihomo()` detects if Clash binary is Mihomo Meta variant (subscribe:clash.py:720)

**subscribe/subconverter.py**

- Wraps the subconverter binary to convert proxies between formats
- `CONVERT_TARGETS` list defines supported output formats (clash, v2ray, singbox, mixed, clashr, quan, quanx, loon, ss, sssub, ssd, ssr, surfboard, surge) (subscribe:subconverter.py:16)
- `generate_conf()` generates INI config files (`generate.ini`) for subconverter execution (subscribe:subconverter.py:55)
- `convert()` executes subconverter binary with `-g` flag (subscribe:subconverter.py:124)
- Thread-safe file operations using `FILE_LOCK` (subscribe:subconverter.py:14)

**subscribe/workflow.py**

- `TaskConfig` dataclass with all task parameters (domain, sub, coupon, rename rules, etc.) (subscribe:workflow.py:20)
- `execute()` runs a single collection task - registers, fetches subscription, parses proxies (subscribe:workflow.py:86)
- `executewrapper()` provides tuple return for multi-threading (subscribe:workflow.py:137)
- `merge_config()` consolidates duplicate tasks (subscribe:workflow.py:215)
- `refresh()` updates remote config and marks invalid subscriptions (subscribe:workflow.py:274)
- `standard_sub()` validates subscription URL format (subscribe:workflow.py:355)

**subscribe/push.py**

- Abstract `PushTo` base class for publishing results (subscribe:push.py:19)
- `PushToGist` implementation uploads to GitHub Gist (subscribe:push.py:387)
- Additional implementations: PushToPasteGG, PushToFarsEE, PushToDevbin, PushToPastefy, PushToDrift, PushToImperial, PushToLocal
- `get_instance()` factory method instantiates appropriate push client based on engine parameter (subscribe:push.py:465)
- Handles file creation/update operations via HTTP API

### Binary Dependencies

Located in platform-specific directories:

- **Clash/Mihomo**: `clash/clash-{darwin,linux}-{amd64,arm64}` - Proxy client for validation
- **Subconverter**: `subconverter/subconverter-{darwin,linux}-{amd64,arm64}` - Format converter
- `executable.py` detects platform and selects correct binary (subscribe:executable.py)

### Configuration Files

- `requirements.txt` - Python dependencies: PyYAML, tqdm, geoip2, pycryptodomex, fofa-hack
- `data/domains.txt` - Airport domain list with format: `domain@#@#coupon@#@#invite_code`
- `data/subscribes.txt` - Collected subscription URLs
- `data/valid-domains.txt` - Verified working domains
- `subconverter/base/*.{yml,json}` - Template configs for different proxy formats
- `subconverter/generate.ini` - Temporary config file for subconverter (auto-generated, cleaned up)

## Common Commands

### Development

```bash
# Install dependencies
pip3 install -r requirements.txt

# Basic collection (with validation)
python3 -u subscribe/collect.py

# Quick collection (skip validation)
python3 -u subscribe/collect.py --skip

# Full refresh with overwrite
python3 -u subscribe/collect.py --all --overwrite

# Refresh existing subscriptions only
python3 -u subscribe/collect.py --refresh
```

### Command-Line Options

- `-a, --all` - Generate full Clash configuration (not just node list)
- `-s, --skip` - Skip proxy usability checks via Clash
- `-o, --overwrite` - Overwrite existing domains.txt with newly crawled sites
- `-r, --refresh` - Only update existing subscriptions, skip new registrations
- `-c, --chuck` - Discard sites requiring human verification (captcha)
- `-e, --easygoing` - Use Gmail aliases for whitelisted mailboxes
- `-n, --num NUM` - Thread count (default: 64)
- `-d, --delay MS` - Max allowed proxy delay in ms (default: 5000)
- `-t, --targets` - Output formats: clash, v2ray, singbox, mixed, etc. (default: clash, v2ray, singbox)
- `-u, --url URL` - Test URL (default: <https://www.google.com/generate_204>)
- `-v, --vitiate` - Ignore default proxies filter rules
- `-g, --gist USER/GIST_ID` - GitHub Gist upload destination (or use GIST_LINK env var)
- `-k, --key TOKEN` - GitHub Personal Access Token (or use GIST_PAT env var)
- `-f, --flow GB` - Minimum remaining traffic in GB
- `-l, --life HOURS` - Minimum remaining lifetime in hours
- `-i, --invisible` - Don't show check progress bar
- `-p, --pages NUM` - Max page number when crawling telegram
- `-y, --yourself URL` - URL to custom airport list (or use CUSTOMIZE_LINK env var)

### Docker

```bash
# Build
docker buildx build --platform linux/amd64 -f Dockerfile -t wzdnzd/aggregator:tag .

# Run (requires GIST_PAT and GIST_LINK env vars)
docker run -e GIST_PAT=xxx -e GIST_LINK=user/gistid wzdnzd/aggregator:tag
```

### GitHub Actions

**Workflow: `.github/workflows/collect.yaml`**

- 计划任务：每周一 00:00（上海时间）
- 必需的 secrets：`GIST_PAT`, `GIST_LINK`
- 可选配置：`CUSTOMIZE_LINK` 自定义机场列表，`ENABLE_SPECIAL_PROTOCOLS` 变量
- 执行命令：`python -u subscribe/collect.py --all --overwrite --skip`

**Workflow: `.github/workflows/refresh.yaml`**

- 计划任务：每 2 小时执行一次
- 必需的 secrets：`GIST_PAT`, `GIST_LINK`
- 执行命令：`python -u subscribe/collect.py --all --refresh --overwrite --skip`
- 用途：刷新现有订阅，不注册新账户

**Workflow: `.github/workflows/process.yaml`**

- 计划任务：每天 03:05 和 11:05（上海时间）
- 必需的 secrets：`SUBSCRIBE_CONF`, `PUSH_TOKEN`
- 可选变量：`REACHABLE`, `SKIP_ALIVE_CHECK`, `SKIP_REMARK`, `WORKFLOW_MODE`, `ENABLE_SPECIAL_PROTOCOLS`
- 执行命令：`python -u subscribe/process.py --overwrite`
- 用途：基于配置文件处理订阅聚合（更高级的配置模式）

## Important Implementation Details

### Multi-threading

- Uses `utils.multi_thread_run()` throughout for concurrent operations
- Default thread pool: 64 threads (configurable via `-n, --num` option)
- Progress bars via `tqdm` (disable with `-i, --invisible`)

### Proxy Validation Flow

1. `assign()` loads existing subscriptions and crawls new domains (subscribe:collect.py:35)
2. `workflow.executewrapper()` runs registration and parsing in parallel (subscribe:collect.py:244)
3. `clash.generate_config()` creates Clash config with all proxies (subscribe:collect.py:258)
4. Start Clash binary as subprocess (subscribe:collect.py:264)
5. `clash.check()` tests each proxy via Clash external controller API (subscribe:collect.py:281)
6. Terminate Clash process (subscribe:collect.py:289)
7. Filter results based on success masks (subscribe:collect.py:293)
8. Convert to multiple formats via `subconverter.generate_conf()` and `subconverter.convert()` (subscribe:collect.py:336)
9. Upload to Gist via `push.PushToGist` (subscribe:collect.py:398)

### Subscription Formats

- Base64-encoded proxy lists
- Clash YAML configs (parsed via PyYAML)
- Singbox JSON configs
- Individual proxy links (ss://, vmess://, trojan://, ssr://, vless://, hysteria://, hysteria2://, tuic://, snell://)
- Special protocols (vless, hysteria, hysteria2, tuic) enabled when using Mihomo and `ENABLE_SPECIAL_PROTOCOLS` env var

### File Naming Conventions

- Airport task names use `utils.random_chars(length=8)` or `crawl.naming_task()` for anonymity (subscribe:crawl.py:1358)
- Subconverter output follows pattern: `{target}.{extension}` via `subconverter.get_filename()` (e.g., `clash.yaml`, `v2ray.txt`, `singbox.json`)
- Data files stored in `data/` directory
- Temporary files cleaned up via `workflow.cleanup()` (subscribe:collect.py:408)

### Error Handling

- Retry mechanism in `TaskConfig.retry` (default: 3) (subscribe:workflow.py:40)
- Expired subscriptions tracked and removed via `crawl.check_status()` (subscribe:crawl.py:1237)
- Invalid domains marked in remote config with `errors` counter
- Logger outputs throughout codebase (subscribe:logger.py)

## Adding New Crawl Plugins

Create a new file in `subscribe/scripts/`:

```python
# subscribe/scripts/mysite.py
def mysite_crawler(params):
    """
    Called via execute_script() from crawl.py

    Returns list of dicts with format:
    [
        {
            "name": "task_name",
            "sub": "subscription_url",  # or list of URLs
            "origin": "SOURCE_NAME",
            "push_to": ["group1", "group2"],
            # optional fields: rename, exclude, include, coupon, invite_code, etc.
        }
    ]

    Or for single proxy links, return:
    {
        "subscription_url": {
            "origin": "SOURCE_NAME",
            "push_to": ["group1", "group2"],
            "proxies": ["ss://...", "vmess://..."]  # list of proxy links
        }
    }
    """
    pass
```

Register in crawl configuration JSON/YAML by adding to `scripts` section with format: `filename#function_name` (e.g., `mysite#mysite_crawler`).

## Environment Variables

**collect.py 相关:**

- `GIST_PAT` - GitHub Personal Access Token，用于 Gist 上传
- `GIST_LINK` - 格式：`username/gist_id`
- `CUSTOMIZE_LINK` - 自定义机场列表的 URL 或文件名
- `ENABLE_SPECIAL_PROTOCOLS` - 启用特殊协议（vless, hysteria, hysteria2, tuic）(airport.py:737)
- `ALLOW_SINGLE_LINK` - 设置为 `"true"` 以爬取单个代理链接 (crawl.py:69)
- `TZ` - 时区（GitHub Actions 默认：Asia/Shanghai）
- `TRACE_ENABLE` - 设置为 "true" 或 "1" 启用 HTTP 请求追踪 (airport.py:469)

**process.py 相关:**

- `SUBSCRIBE_CONF` - 订阅聚合配置文件的 URL 或路径（JSON/YAML 格式）
- `PUSH_TOKEN` - 推送服务的访问令牌
- `WORKFLOW_MODE` - 工作模式：0=爬取并聚合（默认），1=仅爬取，2=仅聚合 (crawl.py:363)
- `REACHABLE` - 设置为 "false" 或 "0" 跳过依赖网络的爬取 (crawl.py:366)
- `SKIP_ALIVE_CHECK` - 跳过存活性检查
- `SKIP_REMARK` - 跳过标记无效订阅

## 辅助工具脚本

`tools/` 目录包含独立的实用工具脚本：

- `tools/auto-checkin.py` - 自动签到脚本
- `tools/renewal.py` - 套餐续期工具
- `tools/filter.py` - 代理过滤工具
- `tools/clean.py` - 清理工具
- `tools/scaner.py` - 端口扫描器
- `tools/ip-location.py` - IP 地理位置查询
- `tools/xui.py` - X-UI 面板相关工具
- `tools/purefast.py` - PureFast 相关工具

## 代码风格注意事项

- domains.txt 字段分隔符：`@#@#` (subscribe:collect.py:129)
- airport.py 重命名分隔符：`RENAME_SEPARATOR = "#@&#@"` (subscribe:airport.py:46)
- 重命名正则表达式组分隔符：`RENAME_GROUP_SEPARATOR = "`"` (subscribe:airport.py:49)
- 所有文件操作使用 UTF-8 编码
- 统一使用 `utils.trim()` 进行字符串清理
- 随机字符生成使用 `LETTERS = set(string.ascii_letters + string.digits)` (subscribe:airport.py:52)
