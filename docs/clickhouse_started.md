# 指南

## 系统要求
如果从官方仓库安装，需要确保您使用的是x86_64处理器构架的Linux并且支持SSE 4.2指令集

检查是否支持SSE 4.2：
```bash
grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
```

推荐使用 `Ubuntu` 或者 `Debian` 。终端必须使用 `UTF-8` 编码。

