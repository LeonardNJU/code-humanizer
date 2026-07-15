# code-humanizer

[English](README.md) | 中文

[![Stars](https://img.shields.io/github/stars/LeonardNJU/code-humanizer?style=flat-square)](https://github.com/LeonardNJU/code-humanizer/stargazers) [![License](https://img.shields.io/github/license/LeonardNJU/code-humanizer?style=flat-square)](LICENSE) [![Last commit](https://img.shields.io/github/last-commit/LeonardNJU/code-humanizer?style=flat-square)](https://github.com/LeonardNJU/code-humanizer/commits) [![Issues](https://img.shields.io/github/issues/LeonardNJU/code-humanizer?style=flat-square)](https://github.com/LeonardNJU/code-humanizer/issues)

**[humanizer](https://github.com/blader/humanizer) 管文字,这个管代码。** 一个 agent skill,用来清除仓库里 AI 生成代码的痕迹——coding agent 为了"测试通过"而牺牲"代码库健康"时留下的结构性屎山。

文字版 humanizer 整理的是 AI *写作*的破绽(em-dash、"不是X而是Y"、三段排比)。这里整理的是 AI *写代码*的破绽——16 条 pattern、5 个层级,每条在 [SKILL.md](SKILL.md) 里都有检测信号和 before/after 例子:

### Pattern 目录

**Tier 1 —— 重复与重造**

| # | Pattern | 破绽 |
|---|---------|------|
| 1 | **重复实现已有 helper**(招牌破绽) | 新写的私有函数和 `utils`/兄弟模块里的东西雷同——agent 从 prompt 往外写,不从仓库往里写 |
| 2 | **`_v2` / `_new` / `_impl` 克隆** | `foo` 和 `foo_v2` 同时活着;agent 不敢改原函数,于是真相有了两份 |
| 3 | **重造标准库/已装依赖** | 手写 `groupby`、JSON 深拷贝、手动解析 URL |

**Tier 2 —— 投机性架构**

| # | Pattern | 破绽 |
|---|---------|------|
| 4 | **单实现抽象** | ABC / registry / "可插拔后端",实现恰好一个、注册恰好一次、调用点恰好一处 |
| 5 | **"备将来用"的死代码** | 没有调用点的 helper;docstring 里写着 *flexible、extensible、seamlessly* |
| 6 | **零增值 wrapper** | 函数体就是同参转发一个调用 |
| 7 | **Config/API 蔓延** | 为局部场景新增全局 flag 或公共参数,只有一处在读 |

**Tier 3 —— 防御性屎感**

| # | Pattern | 破绽 |
|---|---------|------|
| 8 | **宽泛吞异常** | `except Exception: return ""`——把崩溃(可见、可调试)变成数据污染(不可见) |
| 9 | **无理由 try-import fallback** | `try: import ujson except ImportError: import json`,没有 benchmark、没有 extras 声明、fallback 路径没有测试 |
| 10 | **属性探测链** | `hasattr`/`getattr`/`isinstance` 梯子,同时接受"dict 或对象或 None" |
| 11 | **偏执重校验** | 对刚构造出来的值 `if x is not None` |

**Tier 4 —— 噪音**

| # | Pattern | 破绽 |
|---|---------|------|
| 12 | **复读注释** | 注释复述下一行代码(`# Join the rows with newlines`) |
| 13 | **模板 docstring** | docstring 就是函数名加空格;*robust、comprehensive、seamless* |
| 14 | **死 import、无用变量、装饰横幅** | 删剩的残留;`# ===== SECTION =====`;遗漏的 debug print |

**Tier 5 —— 测试屎感**(默认只报告)

| # | Pattern | 破绽 |
|---|---------|------|
| 15 | **断言 mock 的测试** | 所有协作对象都被 mock,测试永远不会因真实原因失败 |
| 16 | **琐碎/重复断言** | 断言字面量;同一个 case 换三个名字重测 |

每条发现有 0–4 严重度,其中 **1 = 存在但合理 → 豁免不动**:有文档理由的 fallback、信任边界处的防御代码、真插件系统的 registry、迁移期的 `_v2`——都保留。目录只是这个 skill 的一半,另一半更重要:

## 为什么不只是一份 pattern 清单

现在的模型指着文件看,大部分屎感它自己就认得出来。**它们缺的不是眼力,是纪律。** 我们的基线测试里,一个没装 skill 的 agent 把样本清理得挺漂亮——但顺手把一个公开的报错类型"改进"了(`AttributeError` 换成"更合理的" `ValueError`),整个改动挤在一个没法 review 的大提交里,动手之前测试一次都没跑过。

所以这个 skill 的核心是三条铁律:

1. **行为保全绝对优先**——包括报错类型和时机。潜在 bug 只*报告*,绝不在清理时顺手"修好"。
2. **没有测试就不动代码。** 测试套件是"语义不变"的唯一裁判,没有裁判就只出报告。
3. **一类 pattern 一个 commit**,每步跑完整测试;有行为风险的改动单独放进 `[BEHAVIOR]` 标记的提交。

外加**防误伤条款**:severity 1 = "存在但有理由"(见上)一律豁免。目标是更健康的仓库,不是击杀数。

## 实战记录

首次实测:一个私有 ML 研究仓库,13.4k 行 Python(21 模块的包 + 36 个实验脚本 + 20 个测试文件),大部分由 coding agent 在人工 review 下写成。扫描模式,零修改,扫前扫后 `git status` 全程干净。

**24 条发现——画像非常鲜明。** 防御性屎感(吞异常、try-import、探测链、复读注释):**0 条**。测试屎感:**20 个测试文件全部 0 条**。债务几乎全部是 **Tier-1 复制型**(11 条发现、40+ 处粘贴),而且副本已经开始咬人:

- 一个溯源 helper 被粘进 **22 个 run 脚本里的 20 个,并且已经分叉**——3 份多了环境变量 fallback,另外 17 份没有;
- 一个实验类被整体复制成 "learned" 变体,指标方法**悄悄分道扬镳**——一份检查 3 组干预对,另一份只查 1 组;
- 同一个正交矩阵采样器粘了 4 次,同一个 query dataclass 跨模块逐字粘贴。

4 条发现被判为合理豁免(severity 1):预注册守卫、有意的采样策略对照、带最终 `raise` 的有界重试。36 个脚本里 8 个、21 个模块里 6 个完全干净——干净就报干净。

结论正好印证了这个 skill 的前提:在人工 review 下工作的 agent 不吞异常——**它们只是不停地重写已经存在的东西**。仓库上下文的重复检测,才是真正要害的那一层。

## 安装

把这个目录复制(或 clone)到你的 agent skills 目录:

```bash
# Claude Code
git clone https://github.com/LeonardNJU/code-humanizer ~/.claude/skills/code-humanizer
```

任何能读 `SKILL.md` agent skill 的框架都一样用——整个 skill 就是一个 Markdown 文件,没有构建步骤,没有依赖。

## 用法

```
> 用 code-humanizer 扫一下这个 PR
> deslop pkg/report.py——我批准你直接修
> 这个仓库是 vibe-coding 糊出来的,humanize 它(先只出报告)
```

默认模式是**扫描 → 报告**(findings 表:pattern 编号、严重度 0–4、证据、修法建议、行为风险)。修复模式需要你批准才会进入,一类 pattern 一个 commit,每步测试全绿。

## 边界说明

- 例子是 Python 的;pattern 和工作流是语言无关的(检测信号里的 Python 惯用法按需移植)。
- 这个 skill 清的是*结构性*债,不管格式偏好——那是 linter 的活。
- 需要深度判断的债(治标还是治本)只报告,不自动修。

## 贡献新 pattern

目录今天是 16 条,但永远不会"完工"——agent 总在发明新的屎感。如果你在 vibe-coding 里撞见了没被覆盖的模式,[提个 issue](https://github.com/LeonardNJU/code-humanizer/issues) 附上一段最小 before/after 例子(直接 PR 更好)。新 pattern 进目录的方式和最初 16 条一样:先有真实的失败例子,再立规则。

## Star History

<a href="https://www.star-history.com/?repos=LeonardNJU%2Fcode-humanizer&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=LeonardNJU/code-humanizer&type=date&theme=dark&legend=top-left&sealed_token=PxMzEEMfOA3R97IHVMG0zQqC1xVjuoA3cMbkfPzjtmhwAbL9n3oQVqHhhzNgKOwqwD0CShQpk_ostzzb2_m-8qA_U5IFphEK5su_nHoI1HKzcVq3-8oq8HZGL79TUmv5WtGBpBCqOtdrgrF8_KKxta_MCbS9IscnCEtMju92w8_qG8TtIZw_zZ68xYli" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=LeonardNJU/code-humanizer&type=date&legend=top-left&sealed_token=PxMzEEMfOA3R97IHVMG0zQqC1xVjuoA3cMbkfPzjtmhwAbL9n3oQVqHhhzNgKOwqwD0CShQpk_ostzzb2_m-8qA_U5IFphEK5su_nHoI1HKzcVq3-8oq8HZGL79TUmv5WtGBpBCqOtdrgrF8_KKxta_MCbS9IscnCEtMju92w8_qG8TtIZw_zZ68xYli" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=LeonardNJU/code-humanizer&type=date&legend=top-left&sealed_token=PxMzEEMfOA3R97IHVMG0zQqC1xVjuoA3cMbkfPzjtmhwAbL9n3oQVqHhhzNgKOwqwD0CShQpk_ostzzb2_m-8qA_U5IFphEK5su_nHoI1HKzcVq3-8oq8HZGL79TUmv5WtGBpBCqOtdrgrF8_KKxta_MCbS9IscnCEtMju92w8_qG8TtIZw_zZ68xYli" />
 </picture>
</a>

## 致谢

Pattern 目录的形式受 [blader/humanizer](https://github.com/blader/humanizer) 启发。债务分类学来自一个关于 agent-induced technical debt 的研究项目(correctness-equivalent patch 分析);三条铁律来自看着能干的 agent 在没有它们时翻车的实录。

## 许可证

[MIT](LICENSE) —— 随便用:fork、改写、把目录搬进你团队的 skills 目录、塞进商业产品里发布,都可以;唯一的义务是保留许可证声明。如果它帮你的仓库躲掉了一个 `_v2`,欢迎点个 star——但从来不是必须的。

## 社区

所有反馈与新 pattern 报告统一走 [GitHub issues](https://github.com/LeonardNJU/code-humanizer/issues)。

介绍帖:[linux.do](https://linux.do/t/topic/2590134/) · [NJU-AIA 论坛](https://forum.nju-aia.com/t/topic/265)

友链:[linux.do](https://linux.do)
