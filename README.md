# WorkOS-Agent

## 总设计：
1）入口层：聊天、CLI、API 都只进入同一个 Agent Loop。
2）Agent Loop：只负责任务理解、工具选择、状态决策，不等待长任务。
3）工具层：分两类——即时工具（Git、filesystem、CLI、browser）和后台工具（translation_worker、ocr_worker、excel_worker）。前者同步返回，后者只返回 accepted + task_id。
4）后台任务协议：run_in_background: true 时，由 Task Manager 起子进程/容器 Worker；Worker 持续写 status.json、heartbeat.json、summary.json、result_manifest.json。
5）事件总线：Worker 退出时，无论成功失败，都向消息队列推一条 task-notification；Agent Loop 收到后读取 summary/result_manifest，再生成对你的最终回复。这个模式与你说的三步完全一致，而且和 Claude Code 的 hook/notification、长任务提醒、持久线程、定时任务方向是一致的。
6）监控层：Cron 不负责跑业务，只负责巡检；Watchdog 定时检查心跳、进度时间、退出码、失败率，发现“未退出但卡死”时推 task-alert。
7）安全层：浏览器与执行环境尽量放在隔离容器/VM，限制域名、动作白名单；购买、删除、登录等高影响动作必须 HITL。
8）扩展层：所有外部系统优先经 MCP/连接器接入，这样你以后加 Figma、文档、网页、内部服务时，不需要改 Agent 核心。

未来 3 年最可能的演化：你的 WorkOS-Agent 会从“单机子进程后台任务”升级到“本地 Worker 池”，再到“分布式专职子代理”；Agent Loop 仍只做编排与沟通，真正执行由专业 Worker 完成。这条路长期最有维护价值，因为模型、OCR、翻译器、浏览器框架都能换，但 任务协议 + 事件总线 + 产物清单 不需要重写。
一句话定版：Git + CLI + Filesystem + Browser 是即时执行面；Background Worker + Message Queue + Watchdog 是长程执行面；Agent Loop 是统一大脑。
