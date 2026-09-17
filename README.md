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

`POST /clocks/:id/retests` 每次复测**必须**在请求体中显式携带 `adjustmentId`，且该编号必须满足：

1. 编号存在且属于该钟表；
2. 编号对应该钟表**最近一次**调校（以 `createdAt` 最新者为准）。

出现以下任一情况返回 **409 Conflict**，且**不写入**任何复测记录：

- 编号缺失（未传、为空）；
- 编号不存在或属于其他钟表；
- 编号不是该钟表最近一次调校（需要先补做调校，再引用最新调校编号复测）。

只有校验通过并成功落库的复测才会改变钟表的合格状态；最近一次复测未通过的钟表继续出现在 `GET /clocks/not-qualified` 中。规则上线前 `adjustmentId` 为空的历史复测不做追溯改造，仍可通过历史与列表接口正常查询。

## 闭环示例

```bash
curl http://127.0.0.1:3021/clocks/not-qualified
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/retests \
  -H 'Content-Type: application/json' \
  -d '{"adjustmentId":"adjustment_demo","dailyRateSeconds":12,"amplitude":252,"note":"复测进入目标范围"}'
```

引用失效编号（缺失 / 属于其他钟表 / 已不是最近一次调校）时返回 409，例如：

```bash
curl -i -X POST http://127.0.0.1:3021/clocks/clock_demo/retests \
  -H 'Content-Type: application/json' \
  -d '{"dailyRateSeconds":12,"amplitude":252}'
# HTTP/1.1 409 Conflict  {"error":"复测必须引用该钟表最近一次调校编号，编号缺失"}
```
