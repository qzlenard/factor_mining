# factor_mining
dfq-gp-miner/

├─ requirements.txt
│ 
├─ config.py
│  └─ 全局配置：数据源、股票池、评估频率、窗口参数、GP超参、缓存预算、日志级别、并行worker数等
├─ run_mine.py
│  └─ 主入口：增量拉数→构建/更新面板→(可选)物化缓存→跑GP round/gen→更新因子库→落盘+滚动维护+checkpoint
├─ run_report.py
│  └─ 研究复核入口：对“因子库快照”生成非交易报告（IC统计、分年、衰减、覆盖率、稳定性），落盘 out/reports
├─ docs/
│  ├─ api_contract.csv
│  │  └─ 接口契约表（file,name,parameters）；所有“文件子对话”实现前必须对照它
│  ├─ data_contract.md
│  │  └─ 数据契约：字段、dtype、复权口径、对齐规则、缺失/停牌处理、行业与市值来源
│  ├─ expression_language.md
│  │  └─ 表达式语言：语法、算子签名、常数窗口位置限制、非法表达式判定、规范化规则
│  ├─ fitness_spec.md
│  │  └─ 适应度口径：winsorize/zscore、中性化回归、rankIC/ICIR、FAST→FULL策略、惩罚项定义
│  └─ cache_planner_spec.md
│     └─ 缓存规划器口径：AST→DAG、子表达式复用统计、成本模型、预算约束、manifest格式、失效规则
├─ scripts/
│  ├─ download_data.py
│  │  └─ 一次性/增量下载原始数据（OHLCV/参考数据），写入 data/raw 与 data/metadata
│  ├─ build_panel.py
│  │  └─ 从 raw 构建/更新 panel（date×code 矩阵或分块），可选生成基础特征到 data/panel
│  ├─ plan_cache.py
│  │  └─ 对指定表达式集合输出缓存计划（manifest）到 out/cache_plan（不做物化）
│  ├─ materialize_cache.py
│  │  └─ 按 manifest 物化中间量到 data/cache，并输出命中率与节省估计
│  └─ export_library.py
│     └─ 把当前因子库快照导出为独立文件（expression+指标），便于下游复用或人工审查
├─ src/
│  ├─ api/
│  │  ├─ myquant_io.py
│  │  │  └─ 掘金SDK适配层：trade_days、index_members、ohlcv、industry_map、market_cap、(可选)fundamentals
│  │  └─ rate_limit.py
│  │     └─ API节流/重试/超时与错误归类（避免脚本层散落重试逻辑）
│  ├─ calendar.py
│  │  └─ 交易日历工具：交易日列表、前后移、对齐（用于fwd收益、增量处理）
│  ├─ universe.py
│  │  └─ 股票池构建：CSI500成分（可滚动）、过滤ST/上市天数/停牌规则的统一实现
│  ├─ store.py
│  │  └─ 本地数据湖I/O：raw/panel/cache/metadata 的读写、版本哈希、原子写入与锁
│  ├─ features/
│  │  ├─ registry.py
│  │  │  └─ 特征注册表：feature_name → builder/loader；统一元数据与依赖声明
│  │  ├─ base_features.py
│  │  │  └─ 日频基础特征生成（ret/lnret/range/turnover等），只依赖OHLCV与少量参考数据
│  │  └─ intraday_agg.py
│  │     └─ （可选）分钟数据→日内聚合特征；聚合后落盘到日频特征，避免保存分钟全量
│  ├─ ops/
│  │  ├─ registry.py
│  │  │  └─ 算子注册表：op_name → 实现函数/输入输出约束/窗口参数约束/可缓存标记/成本模型标签
│  │  ├─ safe_math.py
│  │  │  └─ 安全数学：safe_div/safe_log/safe_sqrt/clip_inf_nan 等，保证表达式评估不崩
│  │  ├─ elementwise.py
│  │  │  └─ 逐元素算子（+ - * /、abs、log、rank-friendly非线性等）
│  │  ├─ cross_section.py
│  │  │  └─ 截面算子（rank、zscore、winsorize、group-neutral等）
│  │  ├─ time_series.py
│  │  │  └─ 时序算子（rolling mean/std/min/max/corr/cov/ewm/delay/delta等）
│  │  └─ cost_model.py
│  │     └─ 成本模型：给每类算子分配估计计算成本（供缓存规划与GP启发式使用）
│  ├─ expr/
│  │  ├─ ast.py
│  │  │  └─ 表达式AST节点定义（op/children/params/leaf feature/const window等）
│  │  ├─ parser.py
│  │  │  └─ 表达式字符串→AST（函数式嵌套）；同时做基本合法性检查
│  │  ├─ canonicalize.py
│  │  │  └─ AST规范化（交换律排序、常数位置/窗口合法化、语义等价判定基础），用于去重与DAG合并
│  │  ├─ dag.py
│  │  │  └─ 多表达式AST合并为DAG（公共子表达式合并），并生成拓扑执行顺序
│  │  └─ evaluator.py
│  │     └─ DAG执行器：给定(date,codes,panel)批量评估表达式；支持内存缓存与落盘缓存；输出命中率指标
│  ├─ neutralize/
│  │  ├─ preprocess.py
│  │  │  └─ 清洗：winsorize(1%)+zscore；缺失处理；样本覆盖率统计
│  │  └─ xsec_regress.py
│  │     └─ 截面回归工具：Ridge(α=1e-6)残差、中性化、R²/N/病态提示；失败模式统一化
│  ├─ fitness/
│  │  ├─ rank_ic.py
│  │  │  └─ rankIC/ICIR计算与对齐（暴露t vs forward return t→t+fwd）；可输出分年统计
│  │  ├─ penalties.py
│  │  │  └─ 惩罚项：长度惩罚、相关性惩罚、max_corr阈值归零规则
│  │  └─ evaluator.py
│  │     └─ 适应度评估器：FAST→FULL两阶段；产出统一metrics字典（ic、n、miss、r2、penalties等）
│  ├─ gp/
│  │  ├─ operators.py
│  │  │  └─ GP树操作：随机生成、变异、交叉、hoist等；遵守窗口常数位置限制与非法表达式过滤
│  │  ├─ selection.py
│  │  │  └─ 选择策略：tournament、精英保留、亲子竞争；输出每代统计
│  │  ├─ dedup.py
│  │  │  └─ 去重：基于canonical hash；防止同构表达式污染种群
│  │  ├─ corr_control.py
│  │  │  └─ 相关性控制：同代/精英集相关性惩罚；max_corr超阈值直接淘汰；外部筛选所需统计
│  │  ├─ factor_library.py
│  │  │  └─ 因子库维护：跨轮合并候选→按fitness排序→贪心去相关入库→快照落盘与版本管理
│  │  └─ evolution.py
│  │     └─ GP主循环：run_round/run_generation；驱动评估、选择、生成下一代；写checkpoint与out/candidates
│  ├─ cache_plan/
│  │  ├─ planner.py
│  │  │  └─ 缓存规划器：表达式集合→AST/DAG→子表达式频次+成本+存储估计→输出manifest
│  │  └─ materialize.py
│  │     └─ 缓存物化：按manifest逐项计算并落盘；记录耗时、命中率、失败项与失效规则
│  ├─ parallel/
│  │  ├─ executor.py
│  │  │  └─ 并行端口抽象：Executor.map/submit接口定义；业务层只依赖此接口
│  │  └─ local_pool.py
│  │     └─ 本机多进程实现（spawn兼容），确保不复制巨型面板：worker按key从store读取
│  └─ utils/
│     ├─ logging.py
│     │  └─ 统一日志：step/loop_progress/eta/metrics；控制台+文件；绑定run_id/round/gen上下文
│     ├─ fileio.py
│     │  └─ 原子写入、目录保障、读写安全封装、append_with_rolloff（滚动保留）
│     ├─ state.py
│     │  └─ manifest/checkpoint/锁：load/save_manifest、with_file_lock、checkpoint读写与恢复
│     ├─ filters.py
│     │  └─ 可交易样本过滤：停牌/涨跌停/异常值过滤；输出覆盖率统计（供fitness与日志）
│     └─ numerics.py
│        └─ 数值工具：winsorize/zscore/clip_inf_nan/相关性计算/安全标准化等（跨模块复用）
├─ tests/
│  ├─ test_parser.py
│  │  └─ 表达式解析与canonicalize单测（等价表达式判等、非法表达式拦截）
│  ├─ test_ops.py
│  │  └─ 算子正确性与边界条件（缺失、inf、窗口、对齐）
│  ├─ test_fitness.py
│  │  └─ fitness链路单测（清洗→中性化→rankIC→惩罚项），含样本不足/缺失过高失败模式
│  ├─ test_cache_plan.py
│  │  └─ 缓存规划与manifest稳定性（同输入产生同输出、预算约束生效）
│  └─ smoke_end_to_end.py
│     └─ 端到端烟测：小日期×小股票池跑一轮一代，验证落盘与checkpoint可恢复
├─ data/
│  ├─ raw/       └─ 原始数据分区存储（不入git）
│  ├─ panel/     └─ 面板与派生基础特征（不入git）
│  ├─ cache/     └─ 中间量缓存（不入git）
│  └─ metadata/  └─ 数据版本、字段映射、构建参数、hash（可入git但通常不）
├─ out/
│  ├─ libraries/   └─ 因子库快照（表达式+核心指标）
│  ├─ candidates/  └─ 每轮/每代候选与精英摘要（便于回溯）
│  ├─ ts/          └─ 滚动时序记录：fitness_history/corr_history/coverage等（keep_last默认252）
│  ├─ cache_plan/  └─ manifest与物化统计
│  ├─ reports/     └─ 研究复核报告（非交易）
│  └─ logs/        └─ 运行日志文本
└─ state/
   ├─ manifest.json   └─ 已处理日期、round/gen、窗口、版本、随机种子、运行参数快照引用
   ├─ checkpoints/    └─ per-round/per-gen checkpoint（种群、精英、评估摘要、rng state）
   └─ locks/          └─ 文件锁目录（原子写与并发安全）
