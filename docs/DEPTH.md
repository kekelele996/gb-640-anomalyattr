# 深度登记 anomalyattr（指标异常归因与维度下钻）

- 规模：中型（6 个功能模块 / 6 个页面 / 80 条测试）
- 核心规则一：**稳健 z 异常检测**——按「门店 × 渠道」建日序列，取当日之前最近 28 个**同层**历史点（工作日 / 休息日分开，区间左闭右开，不含当日），`z = 0.6745 × (x − 中位数) / MAD`；`|z| ≥ 3.5` 为候选，相邻业务日同向连续 2 天为确认；同层点不足 14 不判；MAD 为 0 时退到平均绝对偏差，仍为 0 则本点不判。
- 核心规则二：**LMDI 量价分解**——`净收入 = 可售房晚 × 出租率 × 均价`，权重用对数平均，`Δrev = Δ可售 + Δ出租率 + Δ均价 + 残差`；残差显式量化到分；任一侧因子为 0 时整组记 `A_ZERO_BASE` 并计入残差。
- 使用者与工作流：蓝湾酒店集团经营分析组 8 人，每天 120 门店的渠道日报在 11:00 前汇总；过去靠肉眼翻表，跌幅只能手工拆到量或价。现在导入当日快照即得到「哪些序列连续异常 + 掉在哪一因子 + 哪个门店贡献最多」，四条账在页面上可直接对平。

## 功能模块

| 模块 | 使用者动作 | 入口 | 核心文件 | 测试 |
|---|---|---|---|---|
| M1 指标接入 | 上传或选样例，看到编码、列映射、逐行诊断 | POST /api/import、POST /api/import/preview、页面 / | `app/modules/m1_ingest/service.py` `app/modules/m1_ingest/reader.py` `app/modules/m1_ingest/decode.py` `app/modules/m1_ingest/columns.py` | `tests/test_ingest.py` |
| M2 日历与归一 | 查看字段定义、试算单行、确认时区与分层 | GET /api/contract、POST /api/contract/preview | `app/modules/m2_calendar/service.py` `app/modules/m2_calendar/calendar_rules.py` `app/modules/m2_calendar/tz.py` `app/modules/m2_calendar/contract.py` | `tests/test_calendar.py` |
| M3 质量隔离 | 看原因码分布、导出隔离行、核对三账 | GET /api/quarantine、GET /api/quarantine.csv | `app/modules/m3_quality/service.py` `app/modules/m3_quality/rules.py` `app/modules/m3_quality/dupes.py` `app/modules/m3_quality/quarantine.py` | `tests/test_quality.py` |
| M4 异常检测 | 看水位时间轴、确认点与候选点明细 | GET /api/anomalies、GET /api/anomalies/timeline、页面 /anomalies | `app/modules/m4_detect/service.py` `app/modules/m4_detect/stratify.py` `app/modules/m4_detect/madz.py` `app/modules/m4_detect/series.py` | `tests/test_detect.py` |
| M5 归因与批次 | 看三因子分解、维度份额、同源重跑 | GET /api/attribution、POST /api/attribution/rerun、GET /api/batches、GET /api/reconcile、页面 /attribution /drilldown /batches | `app/modules/m5_attrib/service.py` `app/modules/m5_attrib/lmdi.py` `app/modules/m5_attrib/dimension.py` `app/modules/m5_attrib/batches.py` | `tests/test_attrib.py` |
| M6 报告与导出 | 看运行清单、导出证据文件 | GET /api/report、GET /api/export/{kind}、页面 /report | `app/modules/m6_export/service.py` `app/modules/m6_export/csv_out.py` `app/modules/m6_export/report_build.py` `app/modules/m6_export/manifest_out.py` | `tests/test_export.py` |

端到端：`tests/test_e2e.py`（脏样例 → 隔离 → 三账 → 检测 → 归因 → 导出，含 4xx 与幂等重跑）。

## 陷阱登记

| TRAP | 现象 | 朴素实现会怎样 | 本项目处理 | 处理位置 | 测试 |
|---|---|---|---|---|---|
| TRAP-01 | 历史导出给的是 GB18030 字节，HTTP 头却声明 UTF-8 | `open()` 崩或整表乱码，列名对不上 | 声明编码只作首个候选，严格解码失败回退 GB18030 并记 `N_CHARSET_FALLBACK` | `app/modules/m1_ingest/decode.py` | `tests/test_ingest.py::test_gb18030_fallback_decodes_chinese_headers` `tests/test_ingest.py::test_naive_utf8_decode_would_fail` |
| TRAP-02 | 自家导出的 CSV 带 UTF-8 BOM | 首列名变 `\ufeffhotel_code`，按列名取值全空 | 解码前剥 BOM，留 `N_BOM_STRIPPED` 提示 | `app/modules/m1_ingest/decode.py` | `tests/test_ingest.py::test_bom_stripped_before_header_mapping` |
| TRAP-06 | `1,234.56`、`１２３４.５０`、`980元/间夜` | `float()` 抛错，或把全角串当字符串 | NFKC + 去千分位 + 剥单位后缀，再 Decimal 化 | `app/modules/m2_calendar/service.py` | `tests/test_calendar.py::test_thousands_and_fullwidth_numbers_normalized` `tests/test_calendar.py::test_unit_suffix_stripped` |
| TRAP-07 | 房价列混着 `N/A`、`-`、`未知`、空串 | 被当成 0 参与合计，均价被拉低 | 哨兵清单集中定义，返回三态 `None`，落 `Q_SENTINEL` 隔离 | `app/config.py`、`app/modules/m2_calendar/service.py` | `tests/test_calendar.py::test_sentinel_is_three_state_not_zero` |
| TRAP-11 | 同一快照写成 `2026-11-03T22:00:00Z` 与 `2026-11-04T06:00:00+08:00` | 两个写法落进不同业务日，序列多一个或少一个点 | 先折算 UTC，再取东八区本地日；无偏移按 +08:00 | `app/modules/m2_calendar/tz.py` | `tests/test_calendar.py::test_timezone_offsets_collapse_to_same_business_day` |
| TRAP-13 | 补报数据在 3 天后才到，业务日已落后水位 | 直接丢弃，跌幅被抹平 | 迟到点仍计入有效点，标 `late_flag`；超过补报窗口另记 `L_OUT_OF_WINDOW` | `app/modules/m3_quality/rules.py` | `tests/test_quality.py::test_late_arrival_is_kept_and_marked` |
| TRAP-14 | 同键多条快照，其中一对快照时间相同而数值不同 | 后写覆盖，冲突被悄悄藏起 | 排序键 (快照瞬时, 行号)；同快照时间数值不同整键隔离 `Q_DUP_CONFLICT` | `app/modules/m3_quality/dupes.py` | `tests/test_quality.py::test_duplicate_snapshot_later_wins` `tests/test_quality.py::test_duplicate_conflict_isolated_both_rows` `tests/test_quality.py::test_naive_last_write_wins_keeps_conflict` |
| TRAP-18 | 0 记录 / 仅表头 / 全空行三种空输入 | 都返回“成功导入 0 行”，下游除零 | 三态分开，各自返回 `E_NO_ROWS` / `E_HEADER_ONLY` / `E_ALL_BLANK`（422） | `app/modules/m1_ingest/columns.py`、`app/modules/m1_ingest/reader.py` | `tests/test_ingest.py::test_empty_header_only_state` `tests/test_ingest.py::test_empty_all_blank_state` `tests/test_ingest.py::test_empty_no_rows_state` |
| TRAP-23 | 基线窗口与对比窗口的端点归属 | 两端都算，跨度多一天，相邻期间重复计入 | 一律左闭右开 `[start, end)`；基期结束 = 当期开始 | `app/modules/m2_calendar/calendar_rules.py`、`app/modules/m4_detect/stratify.py`、`app/modules/m5_attrib/service.py` | `tests/test_calendar.py::test_half_open_span_excludes_end` `tests/test_detect.py::test_half_open_window_excludes_current_day` `tests/test_attrib.py::test_windows_are_half_open_and_touch` |
| TRAP-24 | 三因子各自四舍五入后合计与总差不符；份额凑不到 1 | 报告里差额和因子差 0.01，份额合计 100.01% | 因子用 40 位精度算，残差显式量化到分；份额尾差由「其他」或最大组吸收 | `app/modules/m5_attrib/lmdi.py`、`app/modules/m5_attrib/dimension.py` | `tests/test_attrib.py::test_lmdi_residual_is_explicit_zero` `tests/test_attrib.py::test_dimension_shares_conserve_to_one` |

## 对账规则

- 行账：读入记录 = 采信行 + 被覆盖行 + 隔离行（样例 1016 = 1001 + 4 + 11）
- 点账：网格点 = 有效点 + 隔离点 + 缺失点（样例 1008 = 1001 + 5 + 2）
- 金额账：采信行净收入合计 = 有效点净收入合计（Decimal，2 位）
- 分解账：净收入差 = 可售房晚差 + 出租率差 + 均价差 + 残差
- 四账在 /api/reconcile、/api/report、/batches 页面与 `manifest.json` 导出中逐条给出，差额非 0 显式为「不平」。

## 反例与不变量

- 反例（朴素实现会挂）：`tests/test_quality.py::test_naive_last_write_wins_keeps_conflict`（按行序覆盖把冲突藏掉）、`tests/test_ingest.py::test_naive_utf8_decode_would_fail`、`tests/test_calendar.py::test_naive_float_parse_would_raise`。
- 不变量：`tests/test_attrib.py::test_rerun_is_idempotent`（幂等）、`tests/test_quality.py::test_rows_identity_holds` `tests/test_quality.py::test_points_identity_holds` `tests/test_quality.py::test_revenue_identity_holds`（守恒）、`tests/test_quality.py::test_shuffled_input_gives_same_accounts`（顺序无关）。
- 端到端：`tests/test_e2e.py::test_full_pipeline_end_to_end`。

## 已知不覆盖

- 只支持单文件导入；一次导入多个门店的多份文件需要重复调用 POST /api/import，跨文件合并留到下一期。
- 分层只分工作日 / 休息日两类；春节这类连续长假没有单独建层，节假日表只维护 2026 年。
- 异常检测只在 `revenue_net` 上跑；`occ`、`adr` 直接作为分解因子，不单独做异常判定。
- 补报窗口固定 3 天，没有按门店或渠道分别配置；配置项在 `app/config.py` 里。
- 未做权限与多租户；内部 8 人共用一套服务与一个 SQLite 卷。
