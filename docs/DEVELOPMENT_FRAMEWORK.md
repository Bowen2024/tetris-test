# 俄罗斯方块 Python 开发框架

这份文档面向初级工程师和 Coding Agent。先完成可验证的最小阶段，再逐步增加功能。

## 1. 首版目标与边界

首版是一个可在电脑运行的单机俄罗斯方块：

- Python 3.12 + pygame-ce
- 10×20 可见棋盘，无隐藏行
- 七种方块、7-bag、移动、顺时针旋转、软降和硬降
- 碰撞、锁定、消除 1–4 行、计分、等级、暂停、重开和 Game Over
- 方块触底立即锁定，不实现 lock delay
- 旋转后碰撞或越界则取消，不实现 SRS 或 wall kick
- 暂不实现音效、最高分持久化、Hold 和 Ghost Piece

Python + pygame-ce 生成桌面应用，不能直接发布到 GitHub Pages。若以后需要浏览器版本，应另行选择 JavaScript/Canvas 或评估 WebAssembly 打包。

## 2. 项目目录

```text
tetris-test/
├── README.md
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
    ├── test_piece.py
    ├── test_board.py
    ├── test_randomizer.py
    ├── test_scoring.py
    └── test_game.py
```

第二里程碑再增加：

```text
src/tetris/storage.py
tests/test_storage.py
assets/sounds/
```

## 3. 安装与启动

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

验证命令：

```bash
pytest
ruff check .
ruff format --check .
```

`pyproject.toml` 至少包含：

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

## 4. 数据协议

`shapes.py` 只定义方块的局部坐标。每个旋转状态由四个格子组成：

```python
Cell = tuple[int, int]
Rotation = tuple[Cell, Cell, Cell, Cell]

SHAPES: dict[str, tuple[Rotation, ...]] = {
    "O": (
        ((0, 0), (1, 0), (0, 1), (1, 1)),
    ),
    # I、J、L、S、T、Z 按相同结构补齐
}
```

`Piece` 至少包含：

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Piece:
    kind: str
    rotation: int
    x: int
    y: int
```

坐标约定：

- `x` 从左向右增加，`y` 从上向下增加。
- 实际格子坐标等于 Piece 的 `x/y` 加当前旋转状态的局部坐标。
- 方块出生时，局部包围盒顶部位于 `y = 0`，水平方向居中。
- 新方块出生时若 `Board.can_place(piece)` 为 `False`，游戏立即结束。
- `board.py`、`piece.py` 和 `shapes.py` 不得导入 pygame。

## 5. 模块职责

- `main.py`：初始化 pygame，建立窗口、时钟和 Game，启动主循环。
- `config.py`：保存棋盘大小、格子尺寸、FPS、颜色和速度曲线。
- `shapes.py`：定义七种方块及其旋转局部坐标。
- `piece.py`：定义不可变 Piece 数据，返回移动或旋转后的新对象。
- `board.py`：负责碰撞判断、锁定方块和消除满行，不处理输入和绘图。
- `randomizer.py`：实现 7-bag，并允许传入随机数生成器以便测试。
- `scoring.py`：负责消行计分、等级和下落间隔。
- `input_handler.py`：只把 pygame 事件映射为 GameAction。
- `game.py`：唯一的规则协调者，接收动作、更新状态、生成方块并判断结束。
- `renderer.py`：读取状态并绘制，不修改 Board 或 Piece。

## 6. 游戏规则

### 6.1 移动

先生成候选 Piece。只有 `Board.can_place(candidate)` 为 `True` 时才替换当前 Piece，否则保持原状。

### 6.2 旋转

首版只支持顺时针旋转：

1. 计算 `rotation = (rotation + 1) % rotation_count`。
2. Piece 的 `x` 和 `y` 保持不变。
3. 候选方块越界或碰撞时取消旋转。
4. 不尝试左右移动补偿。

### 6.3 下落与锁定

- 自动下落或软降时，下一格合法则移动。
- 下一格不合法时立即锁定。
- 硬降反复向下移动到最后合法位置，然后立即锁定。
- 锁定顺序是：消行 → 计分 → 生成下一方块 → 判断 Game Over。

### 6.4 时间累计

`Game.update(delta_seconds)` 必须使用累计器：

```python
self.fall_accumulator += delta_seconds

while self.fall_accumulator >= self.fall_interval:
    self.fall_accumulator -= self.fall_interval
    self.step_down()

    if self.state != GameState.PLAYING:
        break
```

暂停期间不累计下落时间，继续时不补算暂停时间。

### 6.5 输入边界

`InputHandler` 只返回动作，不直接修改规则状态：

```python
from enum import Enum, auto


class GameAction(Enum):
    MOVE_LEFT = auto()
    MOVE_RIGHT = auto()
    ROTATE_CLOCKWISE = auto()
    SOFT_DROP = auto()
    HARD_DROP = auto()
    TOGGLE_PAUSE = auto()
    RESTART = auto()
    QUIT = auto()
```

只有 `Game.apply_action(action)` 可以修改游戏状态。

## 7. 推荐开发顺序

1. 项目可启动：配置 `pyproject.toml`、命令入口和空窗口。
2. Piece / Shapes：完成数据协议、七种方块和顺时针旋转。
3. Board + 测试：实现边界、重叠、锁定和消行。
4. Randomizer + 测试：实现 7-bag 和固定随机种子。
5. Game 状态机：实现 Ready、Playing、Paused 和 Game Over。
6. 输入：把事件映射成动作，再由 Game 执行。
7. Renderer：绘制棋盘、当前方块、Next 和状态信息。
8. 计分与等级：实现消行计分和速度变化。
9. 第二里程碑：增加最高分、音效和视觉优化。

## 8. 必须通过的测试

- [ ] 方块不能越过左、右和底部边界。
- [ ] 方块不能与已锁定格子重叠。
- [ ] 非法移动和非法旋转保持原状态。
- [ ] 一次可以正确消除 1、2、3、4 行。
- [ ] 消行后上方格子正确下移。
- [ ] 每个 7-bag 恰好包含七种不同方块。
- [ ] 使用固定随机种子时顺序可重复。
- [ ] 硬降停在最后合法位置并立即锁定。
- [ ] 新方块出生位置冲突时进入 Game Over。
- [ ] 暂停时 `update` 不改变棋盘和方块位置。
- [ ] 较大的 delta time 能补足多个下落步，不因低帧率变慢。
- [ ] 计分和等级切换符合配置。

## 9. Coding Agent 的第一个任务

> 只完成“项目基线”：创建上述精简目录，编写 `pyproject.toml`、`.gitignore`、`README.md` 和最小 pygame 空窗口，配置 `tetris` 命令入口。暂不实现方块或游戏规则。完成后运行 `pytest`、`ruff check .` 和 `ruff format --check .`，并报告文件变化、命令输出和剩余风险。

