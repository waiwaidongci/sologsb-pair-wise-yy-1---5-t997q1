# 机械钟表擒纵调校API

纯后端零依赖Node服务，使用 `data/db.json` 持久化钟表档案、调校记录和复测记录。

## 启动

```bash
PORT=3021 node server.js
```

## 主要接口

- `GET /health`
- `GET /clocks`
- `POST /clocks`
- `GET /clocks/not-qualified`
- `GET /clocks/:id/history`
- `POST /clocks/:id/adjustments`
- `POST /clocks/:id/retests`
- `GET /clocks/:id/latest-retest`
- `GET /adjustments?clockId=`
- `GET /retests?clockId=&qualified=`

## 复测可追溯规则

`POST /clocks/:id/retests` 每次复测**必须**在请求体中携带该钟表**最近一次**调校的编号 `adjustmentId`，服务端在写入前依次校验：

1. 编号缺失 → `409`，不写入复测记录；
2. 编号不存在或属于其他钟表 → `409`，不写入复测记录；
3. 编号存在且属于该钟表，但不是最近一次调校 → `409`，不写入复测记录。

只有校验通过并成功落库的复测才会改变钟表的合格状态：合格状态取该钟表**最新一条已落库复测**的结论，最新复测未通过时该钟表继续出现在 `GET /clocks/not-qualified` 中；被拒绝的复测不会产生任何记录、也不影响状态。

规则上线前产生的历史复测可能没有调校编号（`adjustmentId` 为 `null`），这些记录仍然保留，可通过历史与列表接口正常查询。

## 闭环示例

```bash
# 1. 查询未合格钟表
curl http://127.0.0.1:3021/clocks/not-qualified

# 2. 新增一次调校，拿到调校编号
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/adjustments \
  -H 'Content-Type: application/json' \
  -d '{"currentDailyRateSeconds":31,"direction":"慢针方向","amount":"游丝快慢针再向慢侧微调0.2格"}'

# 3. 复测必须引用上一步返回的最近一次调校编号
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/retests \
  -H 'Content-Type: application/json' \
  -d '{"adjustmentId":"<最近一次调校编号>","dailyRateSeconds":12,"amplitude":252,"note":"复测进入目标范围"}'
```
