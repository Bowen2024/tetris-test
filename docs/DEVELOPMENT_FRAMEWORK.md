# 俄罗斯方块 Python 开发框架

这份文档是首版游戏的完整实现规格，面向初级工程师和 Coding Agent。实现者不需要读取聊天记录，也不应自行补充未定义的核心规则。

## 1. 首版目标与边界

首版是一个可在电脑运行的单机俄罗斯方块：

- Python 3.12 + pygame-ce
- 10×20 可见棋盘，无隐藏行
- 七种方块、7-bag、移动、顺时针旋转、软降和硬降
- 碰撞、锁定、消除 1–4 行、计分、等级、暂停、重开和 Game Over
- 方块触底立即锁定，不实现 lock delay
- 旋转后碰撞或越界则取消，不实现 SRS 或 wall kick
- 暂不实现音效、最高分持久化、Hold 和 Ghost Piece

Python + pygame-ce 生成桌面应用，不能直接发布到 GitHub Pages。浏览器版本不属于本项目首版范围。

## 2. 项目目录

```text
tetris-test/
├── README.md
├── AGENTS.md
├── pyproject.toml
├── .gitignore
├── src/
│   └── tetris/
│       ├── __init__.py
│       ├── main.py
│       ├── config.py
│       ├── shapes.py
│       ├── piece.py
│       ├── board.py
│       ├── randomizer.py
│       ├── scoring.py
│       ├── game.py
│       ├── input_handler.py
│       └── renderer.py
└── tests/
    ├── test_main.py
    ├── test_piece.py
    ├── test_board.py
    ├── test_randomizer.py
    ├── test_scoring.py
    ├── test_game.py
    └── test_input_handler.py
```

第二里程碑再增加 `storage.py`、`test_storage.py` 和声音资源。

## 3. 安装、启动与项目配置

macOS 或 Linux：

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -e ".[dev]"
tetris
```

Windows PowerShell 激活环境：

```powershell
.venv\Scripts\Activate.ps1
```

完整验证命令：

```bash
pytest
ruff check .
ruff format --check .
```

`pyproject.toml` 最小配置：

```toml
[build-system]
requires = ["setuptools>=69"]
build-backend = "setuptools.build_meta"

[project]
name = "tetris-test"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["pygame-ce>=2.5,<3"]

[project.optional-dependencies]
dev = ["pytest>=8,<9", "ruff>=0.12,<1"]

[project.scripts]
tetris = "tetris.main:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
pythonpath = ["src"]
testpaths = ["tests"]
```

## 4. 固定常量与棋盘格式

`config.py` 使用以下确定值：

```python
BOARD_WIDTH = 10
BOARD_HEIGHT = 20
CELL_SIZE = 30
SIDE_PANEL_WIDTH = 180
FPS = 60

LINE_CLEAR_POINTS = {0: 0, 1: 100, 2: 300, 3: 500, 4: 800}
LINES_PER_LEVEL = 10
INITIAL_FALL_INTERVAL = 0.8
FALL_INTERVAL_STEP = 0.07
MIN_FALL_INTERVAL = 0.1
SOFT_DROP_POINTS_PER_CELL = 1
HARD_DROP_POINTS_PER_CELL = 2
```

棋盘内部格式：

```python
CellValue = str | None
Grid = list[list[CellValue]]
```

- `None` 表示空格。
- 已锁定格保存方块类型字符串：`"I"`、`"J"`、`"L"`、`"O"`、`"S"`、`"T"` 或 `"Z"`。
- `grid[y][x]` 是坐标 `(x, y)` 的内容。
- `grid` 始终有 20 行，每行始终有 10 列。
- Board 初始化时所有格均为 `None`。

## 5. 七种方块的完整坐标

坐标系中 `x` 向右增加，`y` 向下增加。每个旋转状态包含四个局部格子；旋转顺序就是元组中的顺序。

```python
Cell = tuple[int, int]
Rotation = tuple[Cell, Cell, Cell, Cell]

SHAPES: dict[str, tuple[Rotation, ...]] = {
    "I": (
        ((0, 0), (1, 0), (2, 0), (3, 0)),
        ((0, 0), (0, 1), (0, 2), (0, 3)),
    ),
    "J": (
        ((0, 0), (0, 1), (1, 1), (2, 1)),
        ((0, 0), (1, 0), (0, 1), (0, 2)),
        ((0, 0), (1, 0), (2, 0), (2, 1)),
        ((1, 0), (1, 1), (0, 2), (1, 2)),
    ),
    "L": (
        ((2, 0), (0, 1), (1, 1), (2, 1)),
        ((0, 0), (0, 1), (0, 2), (1, 2)),
        ((0, 0), (1, 0), (2, 0), (0, 1)),
        ((0, 0), (1, 0), (1, 1), (1, 2)),
    ),
    "O": (
        ((0, 0), (1, 0), (0, 1), (1, 1)),
    ),
    "S": (
        ((1, 0), (2, 0), (0, 1), (1, 1)),
        ((0, 0), (0, 1), (1, 1), (1, 2)),
    ),
    "T": (
        ((1, 0), (0, 1), (1, 1), (2, 1)),
        ((0, 0), (0, 1), (1, 1), (0, 2)),
        ((0, 0), (1, 0), (2, 0), (1, 1)),
        ((1, 0), (0, 1), (1, 1), (1, 2)),
    ),
    "Z": (
        ((0, 0), (1, 0), (1, 1), (2, 1)),
        ((1, 0), (0, 1), (1, 1), (0, 2)),
    ),
}
```

`Piece` 使用不可变数据对象：

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Piece:
    kind: str
    rotation: int
    x: int
    y: int
```

Piece 必须提供以下纯逻辑操作：

- `cells()`：返回四个实际棋盘坐标。
- `moved(dx, dy)`：返回坐标变化后的新 Piece。
- `rotated_clockwise()`：返回下一旋转状态的新 Piece。

## 6. 出生位置与 Next

新方块始终使用旋转状态 `0`、`y = 0`。计算该状态局部坐标宽度 `piece_width`：

```python
spawn_x = (BOARD_WIDTH - piece_width) // 2
spawn_y = 0
```

I、J、L、S、T、Z 的出生 `x = 3`，O 的出生 `x = 4`。

游戏创建时从 Randomizer 连续取两个类型：第一个作为 `current_piece`，第二个作为 `next_kind`。当前方块锁定后，用 `next_kind` 生成新方块，再取一个新的 `next_kind`。新方块出生时无法放置则进入 Game Over。

## 7. Board 判定规则

`Board.can_place(piece)` 遍历 `piece.cells()`。每个坐标必须同时满足：

```text
0 <= x < 10
0 <= y < 20
grid[y][x] is None
```

任一格不满足条件就返回 `False`。首版没有隐藏行，因此 `y < 0` 也非法。

`Board.lock_piece(piece)` 的调用前提是 `can_place(piece) is True`；它把每个格写为 `piece.kind`。前提不成立时抛出 `ValueError`，不能部分写入。

`Board.clear_full_lines()`：

1. 找出所有不含 `None` 的行。
2. 删除这些满行。
3. 在顶部插入相同数量的全 `None` 行。
4. 保持棋盘尺寸 10×20。
5. 返回消除行数，结果只能是 0 到 4。

## 8. 随机规则

Randomizer 使用 7-bag：

1. 新袋固定包含 I、J、L、O、S、T、Z 各一次。
2. 使用注入的 `random.Random` 实例执行 `shuffle`。
3. 依次取出方块；袋空后再创建并打乱新袋。

构造函数为 `Randomizer(rng: random.Random | None = None)`。测试传入固定种子，例如 `random.Random(42)`；生产环境未传入时自行创建实例。

## 9. 输入映射

InputHandler 只把 pygame 事件转换为动作，不直接修改规则对象：

```python
from enum import Enum, auto


class GameAction(Enum):
    START = auto()
    MOVE_LEFT = auto()
    MOVE_RIGHT = auto()
    ROTATE_CLOCKWISE = auto()
    SOFT_DROP = auto()
    HARD_DROP = auto()
    TOGGLE_PAUSE = auto()
    RESTART = auto()
    QUIT = auto()
```

| pygame 事件或按键 | GameAction | 行为 |
| --- | --- | --- |
| `QUIT` | `QUIT` | 退出程序 |
| `Left` 或 `A` | `MOVE_LEFT` | 左移一格 |
| `Right` 或 `D` | `MOVE_RIGHT` | 右移一格 |
| `Down` 或 `S` | `SOFT_DROP` | 向下一格，成功时加 1 分 |
| `Up`、`W` 或 `X` | `ROTATE_CLOCKWISE` | 顺时针旋转 |
| `Space` | `HARD_DROP` | 落到底并立即锁定，每格加 2 分 |
| `P` 或 `Escape` | `TOGGLE_PAUSE` | Playing 与 Paused 切换 |
| `R` | `RESTART` | Game Over 时重开 |
| `Enter` | `START` | Ready 时开始 |

只处理 `KEYDOWN`。首版不调用 `pygame.key.set_repeat()`：每收到一次 `KEYDOWN` 只执行一次动作，不支持长按连续移动。玩家需要重复按键来重复移动；DAS/ARR 留到后续版本。

## 10. 游戏状态转换

状态枚举固定为 `READY`、`PLAYING`、`PAUSED` 和 `GAME_OVER`。

| 当前状态 | 事件 | 下一状态 | 同时发生的动作 |
| --- | --- | --- | --- |
| READY | START | PLAYING | 清零累计时间，开始自动下落 |
| READY | QUIT | READY | 设置 `is_running = False` |
| PLAYING | TOGGLE_PAUSE | PAUSED | 不再累计下落时间 |
| PLAYING | 出生位置被占用 | GAME_OVER | 停止自动下落 |
| PLAYING | QUIT | PLAYING | 设置 `is_running = False` |
| PAUSED | TOGGLE_PAUSE | PLAYING | 清零累计时间后继续 |
| PAUSED | QUIT | PAUSED | 设置 `is_running = False` |
| GAME_OVER | RESTART | READY | 创建空棋盘，分数、等级和消行数归零，重新准备 current/next |
| GAME_OVER | QUIT | GAME_OVER | 设置 `is_running = False` |

其他状态下未列出的动作全部忽略。Paused、Ready 和 Game Over 时不能移动、旋转或下落。

## 11. 移动、旋转、下落与锁定

### 11.1 水平移动

生成 `current_piece.moved(dx, 0)`。只有候选 Piece 可以放置时才替换当前 Piece，否则保持原状。

### 11.2 顺时针旋转

1. 计算 `rotation = (rotation + 1) % len(SHAPES[kind])`。
2. Piece 的 `x` 和 `y` 保持不变。
3. 候选方块合法时才替换当前 Piece。
4. 候选方块越界或碰撞时取消旋转，不尝试位置补偿。
5. O 只有一个旋转状态，旋转后数据不变。

### 11.3 自动下落与软降

尝试 `moved(0, 1)`。合法则移动；自动下落不加分，软降成功时加 1 分。下一格不合法时，自动下落和软降都立即执行锁定流程。

### 11.4 硬降

循环尝试向下一格，直到下一步不合法。记录成功下降格数 `distance`，增加 `distance * 2` 分，然后立即执行锁定流程。

### 11.5 锁定流程

固定顺序：

1. `board.lock_piece(current_piece)`。
2. `cleared = board.clear_full_lines()`。
3. 更新总消行数和消行分数。
4. 重新计算等级与下落间隔。
5. 用 `next_kind` 创建新 Piece。
6. 从 Randomizer 获取新的 `next_kind`。
7. 新 Piece 无法放置时进入 Game Over。

## 12. 计分、等级与速度

消行分数：

```python
score += LINE_CLEAR_POINTS[cleared] * level
```

单行、双行、三行、四行基础分依次是 100、300、500、800。使用消行发生前的当前等级计算本次分数，然后根据总消行数更新等级。

等级：

```python
level = total_lines // 10 + 1
```

下落间隔：

```python
fall_interval = max(0.1, 0.8 - 0.07 * (level - 1))
```

等级 1 为 0.80 秒/格，等级 2 为 0.73 秒/格，之后每级减少 0.07 秒，最低为 0.10 秒/格。

## 13. 时间累计

仅 `PLAYING` 状态累计时间：

```python
self.fall_accumulator += delta_seconds

while self.fall_accumulator >= self.fall_interval:
    self.fall_accumulator -= self.fall_interval
    self.step_down()

    if self.state != GameState.PLAYING:
        break
```

进入暂停、离开暂停、重开或开始游戏时把 `fall_accumulator` 设为 `0.0`。

## 14. 模块职责与禁止依赖

- `main.py`：初始化 pygame，建立窗口、时钟和 Game，启动主循环。
- `config.py`：保存全部固定常量，不包含可变状态。
- `shapes.py`：保存方块局部坐标，不导入 pygame。
- `piece.py`：实现不可变 Piece 和纯坐标计算，不导入 pygame。
- `board.py`：实现棋盘判定、锁定和消行，不导入 pygame。
- `randomizer.py`：实现可注入随机源的 7-bag，不导入 pygame。
- `scoring.py`：实现纯计分、等级和速度函数，不导入 pygame。
- `input_handler.py`：可以导入 pygame，只负责事件到动作的映射。
- `game.py`：唯一规则协调者，不读取键盘，不直接绘图。
- `renderer.py`：可以导入 pygame，只读取状态并绘制。

## 15. 分阶段开发与验收

每次只完成一个阶段。每阶段都必须运行全部既有测试和 Ruff 检查。

### 阶段 1：项目基线

产物：`pyproject.toml`、`.gitignore`、全部空包目录、最小 `main.py`、README 启动说明，以及 `tests/test_main.py`。

验收：

- `pip install -e ".[dev]"` 成功。
- `tetris` 打开 pygame 窗口并可正常关闭。
- `tests/test_main.py` 至少验证 `from tetris.main import main` 可以成功导入，不在导入时创建窗口或启动循环。
- `pytest`、`ruff check .`、`ruff format --check .` 通过。
- 不实现游戏规则。

### 阶段 2：Shapes 与 Piece

产物：完整 SHAPES、不可变 Piece、`cells/moved/rotated_clockwise`、`test_piece.py`。

验收：

- 七种方块每个旋转状态都恰有四个不重复坐标。
- 移动和旋转返回新对象，不修改原对象。
- O 旋转后格子不变，其他方块按文档顺序循环。

### 阶段 3：Board

产物：空棋盘、`can_place`、`lock_piece`、`clear_full_lines`、`test_board.py`。

验收：

- 左右、顶部、底部越界均拒绝。
- 与锁定格重叠时拒绝。
- 锁定失败不会部分写入。
- 0–4 行消除和顶部补空行正确，尺寸始终为 10×20。

### 阶段 4：Randomizer

产物：可注入 `random.Random` 的 7-bag 和 `test_randomizer.py`。

验收：

- 每连续七次取值恰含七种方块各一次。
- 跨袋取值不丢失、不重复消费。
- 相同种子产生相同序列。

### 阶段 5：计分与 Game 状态机

产物：`scoring.py`、GameState、GameAction、出生/Next、移动、旋转、锁定、计分、等级、时间累计、`test_scoring.py` 和 `test_game.py`。

验收：

- 状态转换严格符合状态表。
- 出生阻塞进入 Game Over。
- 软降、硬降、1–4 行消除计分精确。
- 每 10 行升级，速度符合公式且不低于 0.10 秒。
- 大 delta time 能补足多个下落步。

### 阶段 6：输入

产物：`input_handler.py`、`tests/test_input_handler.py`，以及 Game 接收并执行动作的连接代码。

验收：

- 每个按键只映射到规定动作。
- InputHandler 不修改 Game、Board 或 Piece。
- 单个 `KEYDOWN` 只产生一个动作；不调用 `pygame.key.set_repeat()`。
- 非 PLAYING 状态忽略移动、旋转和下落动作。

### 阶段 7：Renderer 与完整可玩流程

产物：棋盘、当前方块、Next、分数、等级、消行数和状态提示。

验收：

- 可以从 Ready 开始并完成一局。
- 暂停、继续、Game Over 和重开可见且可操作。
- Renderer 不修改规则状态。
- 关闭窗口后进程正常退出。
- 完整自动测试和 Ruff 检查通过，并记录人工试玩结果。

## 16. 必须存在的自动测试

- [ ] 方块不能越过左、右、顶部和底部边界。
- [ ] 方块不能与已锁定格子重叠。
- [ ] 非法移动和非法旋转保持原状态。
- [ ] 一次可以正确消除 1、2、3、4 行。
- [ ] 消行后上方格子正确下移。
- [ ] 每个 7-bag 恰好包含七种不同方块。
- [ ] 使用固定随机种子时序列可重复。
- [ ] 软降成功每格加 1 分。
- [ ] 硬降停在最后合法位置、每格加 2 分并立即锁定。
- [ ] 新方块出生位置冲突时进入 Game Over。
- [ ] 暂停时 update 不改变棋盘和方块位置。
- [ ] 大 delta time 能补足多个下落步。
- [ ] 消行分数、等级和速度符合固定公式。
- [ ] 状态表中所有允许和忽略的转换均被覆盖。

## 17. Coding Agent 的完整任务

> 完成本文档定义的整个俄罗斯方块首版。严格按照阶段 1–7 顺序实施：先完成项目基线并验证，再依次实现 Shapes/Piece、Board、Randomizer、计分与状态机、输入和 Renderer。每个阶段必须满足对应产物与验收条件，运行全部既有 `pytest`、`ruff check .` 和 `ruff format --check .`；验证通过后自动继续下一阶段，不要在阶段 1 停止，也不要等待聊天中再次授权。最终交付必须保证 `tetris` 可启动并完整游玩，所有自动检查通过，并报告文件变化、设计对应关系、命令结果、人工试玩结果和剩余风险。只有遇到无法从本仓库规格解决的真实阻塞时才向用户提问。
