# 使用Koog框架构建人工智能棋手

[在GitHub上打开](https://github.com/JetBrains/koog/blob/develop/examples/notebooks/Chess.ipynb)
[下载.ipynb](https://raw.githubusercontent.com/JetBrains/koog/develop/examples/notebooks/Chess.ipynb)

本教程演示如何使用Koog框架构建智能象棋代理。我们将探索关键概念，包括工具集成、代理策略、内存优化和交互式人工智能决策。

## 您将学到什么

- 如何为复杂游戏建模特定领域的数据结构
- 创建客服代表可用于与环境交互的自定义工具
- 通过内存管理实施高效的代理策略
- 构建具有选择功能的交互式人工智能系统
- 优化回合制游戏的代理性能

## 设置

首先，让我们导入Koog框架并设置我们的开发环境：

```
%useLatestDescriptors
%use koog
```

## 国际象棋领域建模

创建强大的领域模型对于任何游戏AI都是必不可少的。在国际象棋中，我们需要代表玩家、棋子及其关系。让我们首先定义我们的核心数据结构：

### 核心枚举和类型

```
enum class Player {
    White, Black, None;

    fun opponent(): Player = when (this) {
        White -> Black
        Black -> White
        None -> throw IllegalArgumentException("No opponent for None player")
    }
}

enum class PieceType(val id: Char) {
    King('K'), Queen('Q'), Rook('R'),
    Bishop('B'), Knight('N'), Pawn('P'), None('*');

    companion object {
        fun fromId(id: String): PieceType {
            require(id.length == 1) { "Invalid piece id: $id" }

            return entries.first { it.id == id.single() }
        }
    }
}

enum class Side {
    King, Queen
}
```

`Player`枚举代表国际象棋的两面，使用`opponent()`方法可在玩家之间轻松切换。`PieceType`枚举将每个棋子映射到其标准符号字符，从而可以轻松解析国际象棋动作。

`Side`枚举有助于区分国王侧和王后侧城堡动作。

### 棋子和位置建模

```
data class Piece(val pieceType: PieceType, val player: Player) {
    init {
        require((pieceType == PieceType.None) == (player == Player.None)) {
            "Invalid piece: $pieceType $player"
        }
    }

    fun toChar(): Char = when (player) {
        Player.White -> pieceType.id.uppercaseChar()
        Player.Black -> pieceType.id.lowercaseChar()
        Player.None -> pieceType.id
    }

    fun isNone(): Boolean = pieceType == PieceType.None

    companion object {
        val None = Piece(PieceType.None, Player.None)
    }
}

data class Position(val row: Int, val col: Char) {
    init {
        require(row in 1..8 && col in 'a'..'h') { "Invalid position: $col$row" }
    }

    constructor(position: String) : this(
        position[1].digitToIntOrNull() ?: throw IllegalArgumentException("Incorrect position: $position"),
        position[0],
    ) {
        require(position.length == 2) { "Invalid position: $position" }
    }
}

class ChessBoard {
    private val backRow = listOf(
        PieceType.Rook, PieceType.Knight, PieceType.Bishop,
        PieceType.Queen, PieceType.King,
        PieceType.Bishop, PieceType.Knight, PieceType.Rook
    )

    private val board: List<MutableList<Piece>> = listOf(
        backRow.map { Piece(it, Player.Black) }.toMutableList(),
        List(8) { Piece(PieceType.Pawn, Player.Black) }.toMutableList(),
        List(8) { Piece.None }.toMutableList(),
        List(8) { Piece.None }.toMutableList(),
        List(8) { Piece.None }.toMutableList(),
        List(8) { Piece.None }.toMutableList(),
        List(8) { Piece(PieceType.Pawn, Player.White) }.toMutableList(),
        backRow.map { Piece(it, Player.White) }.toMutableList()
    )

    override fun toString(): String = board
        .withIndex().joinToString("\n") { (index, row) ->
            "${8 - index} ${row.map { it.toChar() }.joinToString(" ")}"
        } + "\n  a b c d e f g h"

    fun getPiece(position: Position): Piece = board[8 - position.row][position.col - 'a']
    fun setPiece(position: Position, piece: Piece) {
        board[8 - position.row][position.col - 'a'] = piece
    }
}
```

`Piece`数据类将片段类型与其所有者相结合，在可视表示中对白色片段使用大写字母，对黑色片段使用小写字母。`Position`类通过内置验证封装国际象棋坐标（例如“e4” ）。

## 游戏状态管理

### ChessBoard 实现

`ChessBoard`类管理8 × 8网格和棋子位置。关键设计决策包括：

- * *内部表示* * ：使用可变列表列表进行高效访问和修改
- * *可视化显示* * ： `toString()`方法提供清晰的ASCII表示，包括等级数字和文件字母
- * *位置映射* * ：在国际象棋符号(a1-h8)和内部数组索引之间转换

### ChessGame 逻辑

```
/**
 * Simple chess game without checks for valid moves.
 * Stores a correct state of the board if the entered moves are valid
 */
class ChessGame {
    private val board: ChessBoard = ChessBoard()
    private var currentPlayer: Player = Player.White
    val moveNotation: String = """
        0-0 - short castle
        0-0-0 - long castle
        <piece>-<from>-<to> - usual move. e.g. p-e2-e4
        <piece>-<from>-<to>-<promotion> - promotion move. e.g. p-e7-e8-q.
        Piece names:
            p - pawn
            n - knight
            b - bishop
            r - rook
            q - queen
            k - king
    """.trimIndent()

    fun move(move: String) {
        when {
            move == "0-0" -> castleMove(Side.King)
            move == "0-0-0" -> castleMove(Side.Queen)
            move.split("-").size == 3 -> {
                val (_, from, to) = move.split("-")
                usualMove(Position(from), Position(to))
            }

            move.split("-").size == 4 -> {
                val (piece, from, to, promotion) = move.split("-")

                require(PieceType.fromId(piece) == PieceType.Pawn) { "Only pawn can be promoted" }

                usualMove(Position(from), Position(to))
                board.setPiece(Position(to), Piece(PieceType.fromId(promotion), currentPlayer))
            }

            else -> throw IllegalArgumentException("Invalid move: $move")
        }

        updateCurrentPlayer()
    }

    fun getBoard(): String = board.toString()
    fun currentPlayer(): String = currentPlayer.name.lowercase()

    private fun updateCurrentPlayer() {
        currentPlayer = currentPlayer.opponent()
    }

    private fun usualMove(from: Position, to: Position) {
        if (board.getPiece(from).pieceType == PieceType.Pawn && from.col != to.col && board.getPiece(to).isNone()) {
            // the move is en passant
            board.setPiece(Position(from.row, to.col), Piece.None)
        }

        movePiece(from, to)
    }

    private fun castleMove(side: Side) {
        val row = if (currentPlayer == Player.White) 1 else 8
        val kingFrom = Position(row, 'e')
        val (rookFrom, kingTo, rookTo) = if (side == Side.King) {
            Triple(Position(row, 'h'), Position(row, 'g'), Position(row, 'f'))
        } else {
            Triple(Position(row, 'a'), Position(row, 'c'), Position(row, 'd'))
        }

        movePiece(kingFrom, kingTo)
        movePiece(rookFrom, rookTo)
    }

    private fun movePiece(from: Position, to: Position) {
        board.setPiece(to, board.getPiece(from))
        board.setPiece(from, Piece.None)
    }
}
```

`ChessGame`类协调游戏逻辑并维护状态。值得注意的功能包括：

- * *移动符号支持* * ：接受常规移动、城堡（ 0-0、0-0-0 ）和典当促销的标准国际象棋符号
- * *特殊动作处理* * ：实施过客捕捉和Castling逻辑
- * *回合管理* * ：每次移动后自动在玩家之间交替
- * *验证* * ：虽然它不验证移动合法性（信任AI进行有效移动） ，但它正确处理移动解析和状态更新

`moveNotation`字符串为AI代理提供了可接受的移动格式的清晰文档。

## 与 Koog 框架集成

### 创建自定义工具

```
import kotlinx.serialization.Serializable

class Move(val game: ChessGame) : SimpleTool<Move.Args>(
    argsSerializer = Args.serializer(),
    descriptor = ToolDescriptor(
        name = "move",
        description = "Moves a piece according to the notation:\n${game.moveNotation}",
        requiredParameters = listOf(
            ToolParameterDescriptor(
                name = "notation",
                description = "The notation of the piece to move",
                type = ToolParameterType.String,
            )
        )
    )
) {
    @Serializable
    data class Args(val notation: String) : ToolArgs

    override suspend fun execute(args: Args): String {
        game.move(args.notation)
        println(game.getBoard())
        println("-----------------")
        return "Current state of the game:\n${game.getBoard()}\n${game.currentPlayer()} to move! Make the move!"
    }
}
```

`Move`工具演示了Koog框架的工具集成模式：

1. * *扩展SimpleTool * * ：继承具有类型安全参数处理的基本工具功能
2. * *可序列化的参数* * ：使用Kotlin序列化来定义工具的输入参数
3. * *丰富的文档* * ： `ToolDescriptor`为LLM提供有关工具用途和参数的详细信息
4. * *构造函数参数* * ：将`argsSerializer`和`descriptor`传递给构造函数
5. * *执行逻辑* * ： `execute`方法处理实际移动执行并提供格式化反馈

关键设计方面：
- * *上下文注入* * ：该工具接收`ChessGame`实例，允许其修改游戏状态
- * *反馈循环* * ：返回当前棋盘状态并提示下一个玩家，保持对话流
- * *错误处理* * ：依靠游戏类进行移动验证和错误报告

## Agent 策略设计

### 内存优化技术

```
import ai.koog.agents.core.environment.ReceivedToolResult

/**
 * Chess position is (almost) completely defined by the board state,
 * So we can trim the history of the LLM to only contain the system prompt and the last move.
 */
inline fun <reified T> AIAgentSubgraphBuilderBase<*, *>.nodeTrimHistory(
    name: String? = null
): AIAgentNodeDelegate<T, T> = node(name) { result ->
    llm.writeSession {
        rewritePrompt { prompt ->
            val messages = prompt.messages

            prompt.copy(messages = listOf(messages.first(), messages.last()))
        }
    }

    result
}

val strategy = strategy<String, String>("chess_strategy") {
    val nodeCallLLM by nodeLLMRequest("sendInput")
    val nodeExecuteTool by nodeExecuteTool("nodeExecuteTool")
    val nodeSendToolResult by nodeLLMSendToolResult("nodeSendToolResult")
    val nodeTrimHistory by nodeTrimHistory<ReceivedToolResult>()

    edge(nodeStart forwardTo nodeCallLLM)
    edge(nodeCallLLM forwardTo nodeExecuteTool onToolCall { true })
    edge(nodeCallLLM forwardTo nodeFinish onAssistantMessage { true })
    edge(nodeExecuteTool forwardTo nodeTrimHistory)
    edge(nodeTrimHistory forwardTo nodeSendToolResult)
    edge(nodeSendToolResult forwardTo nodeFinish onAssistantMessage { true })
    edge(nodeSendToolResult forwardTo nodeExecuteTool onToolCall { true })
}
```

`nodeTrimHistory`函数实现了国际象棋游戏的关键优化。由于国际象棋的位置在很大程度上取决于当前的板状态，而不是完整的移动历史记录，因此我们可以通过仅保留以下内容来显着减少令牌的使用：

1. * *系统提示* * ：包含客服代表的核心说明和行为指南
2. * *最新消息* * ：最新的棋盘状态和游戏上下文

目前这种方法用于鉴定帕金森病的新疗法。5
- * *减少令牌消耗* * ：防止对话历史记录呈指数级增长
- * *保持上下文* * ：保留基本的游戏状态信息
- * *提高性能* * ：提示更短，处理速度更快
- * *启用长时间游戏* * ：允许在不达到令牌限制的情况下扩展游戏玩法

CHESS策略展示了Koog基于图的代理架构：

* *节点类型： * *
- `nodeCallLLM` ：处理输入并生成响应/工具调用
- `nodeExecuteTool` ：使用提供的参数执行移动工具
- `nodeTrimHistory` ：如上所述优化会话记忆
- `nodeSendToolResult` ：将工具执行结果发送回LLM

控制流
- * *线性路径* * ：启动→LLM请求→工具执行→历史记录修剪→发送结果
- * *决策要点* * ： LLM回复可以结束对话，也可以触发另一个工具调用
- * *内存管理* * ：每次工具执行后进行历史记录修剪

这种策略确保了高效、有状态的游戏玩法，同时保持了会话的连贯性。

### 设置 AI Agent

```
val baseExecutor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY"))
```

本节初始化我们的OpenAI执行器。`simpleOpenAIExecutor`使用环境变量的API密钥创建到OpenAI API的连接。

XLIII. Informix functions
- 将您的OpenAI API密钥存储在`OPENAI_API_KEY`环境变量中
- 执行者自动处理身份验证和API通信
- 不同的遗嘱执行人类型适用于不同的LLM提供商

### Agent 组装

```
val game = ChessGame()
val toolRegistry = ToolRegistry { tools(listOf(Move(game))) }

// Create a chat agent with a system prompt and the tool registry
val agent = AIAgent(
    executor = baseExecutor,
    strategy = strategy,
    llmModel = OpenAIModels.Chat.O3Mini,
    systemPrompt = """
            You are an agent who plays chess.
            You should always propose a move in response to the "Your move!" message.

            DO NOT HALLUCINATE!!!
            DO NOT PLAY ILLEGAL MOVES!!!
            YOU CAN SEND A MESSAGE ONLY IF IT IS A RESIGNATION OR A CHECKMATE!!!
        """.trimMargin(),
    temperature = 0.0,
    toolRegistry = toolRegistry,
    maxIterations = 200,
)
```

在这里，我们将所有组件组装成一个功能性的国际象棋代理：

* *密钥配置： * *

- * *模型选择* * ：使用`OpenAIModels.Chat.O3Mini`进行高质量的国际象棋比赛
- * *温度* * ：对于确定性的战略移动，设置为0.0
- * *系统提示* * ：精心编制的说明，强调合法行动和正确行为
- * *工具注册表* * ：提供代理对移动工具的访问权限
- * *最大迭代次数* * ：设置为200以允许完成游戏

* *系统提示设计： * *
- 强调MOVE提案的责任
- 禁止幻觉和非法行为
- 将消息传递限制为仅辞职或对方声明
- 创造专注、以游戏为导向的行为

### 运行基本 Agent

```
import kotlinx.coroutines.runBlocking

println("Chess Game started!")

val initialMessage = "Starting position is ${game.getBoard()}. White to move!"

runBlocking {
    agent.run(initialMessage)
}
```

```
Chess Game started!
8 r n b q k b n r
7 p p p p p p p p
6 * * * * * * * *
5 * * * * * * * *
4 * * * * P * * *
3 * * * * * * * *
2 P P P P * P P P
1 R N B Q K B N R
  a b c d e f g h
- ----------------
8 r n b q k b n r
7 p p p p * p p p
6 * * * * * * * *
5 * * * * p * * *
4 * * * * P * * *
3 * * * * * * * *
2 P P P P * P P P
1 R N B Q K B N R
  a b c d e f g h
- ----------------
8 r n b q k b n r
7 p p p p * p p p
6 * * * * * * * *
5 * * * * p * * *
4 * * * * P * * *
3 * * * * * N * *
2 P P P P * P P P
1 R N B Q K B * R
  a b c d e f g h
- ----------------
8 r n b q k b * r
7 p p p p * p p p
6 * * * * * n * *
5 * * * * p * * *
4 * * * * P * * *
3 * * * * * N * *
2 P P P P * P P P
1 R N B Q K B * R
  a b c d e f g h
- ----------------
8 r n b q k b * r
7 p p p p * p p p
6 * * * * * n * *
5 * * * * p * * *
4 * * * * P * * *
3 * * N * * N * *
2 P P P P * P P P
1 R * B Q K B * R
  a b c d e f g h
- ----------------

The execution was interrupted
```

这个基本智能体会自动发挥作用，自动移动。游戏输出显示了AI与自身对抗时的动作和棋盘状态的顺序。

## 高级功能：交互式选择

接下来的部分展示了一种更复杂的方法，用户可以通过从多个AI生成的动作中进行选择来参与AI的决策过程。

### 自定义选择策略

```
import ai.koog.agents.core.feature.choice.ChoiceSelectionStrategy

/**
 * `AskUserChoiceStrategy` allows users to interactively select a choice from a list of options
 * presented by a language model. The strategy uses customizable methods to display the prompt
 * and choices and read user input to determine the selected choice.
 *
 * @property promptShowToUser A function that formats and displays a given `Prompt` to the user.
 * @property choiceShowToUser A function that formats and represents a given `LLMChoice` to the user.
 * @property print A function responsible for displaying messages to the user, e.g., for showing prompts or feedback.
 * @property read A function to capture user input.
 */
class AskUserChoiceSelectionStrategy(
    private val promptShowToUser: (Prompt) -> String = { "Current prompt: $it" },
    private val choiceShowToUser: (LLMChoice) -> String = { "$it" },
    private val print: (String) -> Unit = ::println,
    private val read: () -> String? = ::readlnOrNull
) : ChoiceSelectionStrategy {
    override suspend fun choose(prompt: Prompt, choices: List<LLMChoice>): LLMChoice {
        print(promptShowToUser(prompt))

        print("Available LLM choices")

        choices.withIndex().forEach { (index, choice) ->
            print("Choice number ${index + 1}: ${choiceShowToUser(choice)}")
        }

        var choiceNumber = ask(choices.size)
        while (choiceNumber == null) {
            print("Invalid response.")
            choiceNumber = ask(choices.size)
        }

        return choices[choiceNumber - 1]
    }

    private fun ask(numChoices: Int): Int? {
        print("Please choose a choice. Enter a number between 1 and $numChoices: ")

        return read()?.toIntOrNull()?.takeIf { it in 1..numChoices }
    }
}
```

`AskUserChoiceSelectionStrategy`实现了Koog的`ChoiceSelectionStrategy`接口，使人类能够参与人工智能决策：

关键特性
- * *可自定义显示* * ：用于格式化提示和选项的功能
- * *交互式输入* * ：使用标准输入/输出进行用户交互
- * *验证* * ：确保用户输入在有效范围内
- * *灵活的I/O * * ：可针对不同环境配置打印和读取功能

使用案例
- 人类-人工智能在游戏中的合作
- 人工智能决策的透明度和可解释性
- 培训和调试场景
- 教育演示

### 使用选择增强策略

```
inline fun <reified T> AIAgentSubgraphBuilderBase<*, *>.nodeTrimHistory(
    name: String? = null
): AIAgentNodeDelegate<T, T> = node(name) { result ->
    llm.writeSession {
        rewritePrompt { prompt ->
            val messages = prompt.messages

            prompt.copy(messages = listOf(messages.first(), messages.last()))
        }
    }

    result
}

val strategy = strategy<String, String>("chess_strategy") {
    val nodeCallLLM by nodeLLMRequest("sendInput")
    val nodeExecuteTool by nodeExecuteTool("nodeExecuteTool")
    val nodeSendToolResult by nodeLLMSendToolResult("nodeSendToolResult")
    val nodeTrimHistory by nodeTrimHistory<ReceivedToolResult>()

    edge(nodeStart forwardTo nodeCallLLM)
    edge(nodeCallLLM forwardTo nodeExecuteTool onToolCall { true })
    edge(nodeCallLLM forwardTo nodeFinish onAssistantMessage { true })
    edge(nodeExecuteTool forwardTo nodeTrimHistory)
    edge(nodeTrimHistory forwardTo nodeSendToolResult)
    edge(nodeSendToolResult forwardTo nodeFinish onAssistantMessage { true })
    edge(nodeSendToolResult forwardTo nodeExecuteTool onToolCall { true })
}

val askChoiceStrategy = AskUserChoiceSelectionStrategy(promptShowToUser = { prompt ->
    val lastMessage = prompt.messages.last()
    if (lastMessage is MessagePart.Tool.Call) {
        lastMessage.content
    } else {
        ""
    }
})
```

```
val promptExecutor = PromptExecutorWithChoiceSelection(baseExecutor, askChoiceStrategy)
```

第一种交互式方法使用`PromptExecutorWithChoiceSelection` ，它使用选择选择功能包装基本执行器。自定义显示功能从工具调用中提取移动信息，向用户展示AI想要做什么。

* *架构变更： * *
- * *包装执行器* * ： `PromptExecutorWithChoiceSelection`为任何基本执行器添加了选择功能
- * *上下文感知显示* * ：显示最后一个工具调用内容，而不是完整提示
- * *更高的温度* * ：增加到1.0以获得更多样化的移动选项

### 高级策略：手动选择

```
val game = ChessGame()
val toolRegistry = ToolRegistry { tools(listOf(Move(game))) }

val agent = AIAgent(
    executor = promptExecutor,
    strategy = strategy,
    llmModel = OpenAIModels.Chat.O3Mini,
    systemPrompt = """
            You are an agent who plays chess.
            You should always propose a move in response to the "Your move!" message.

            DO NOT HALLUCINATE!!!
            DO NOT PLAY ILLEGAL MOVES!!!
            YOU CAN SEND A MESSAGE ONLY IF IT IS A RESIGNATION OR A CHECKMATE!!!
        """.trimMargin(),
    temperature = 1.0,
    toolRegistry = toolRegistry,
    maxIterations = 200,
    numberOfChoices = 3,
)
```

高级策略将选择选择直接集成到座席的执行图中：

* *新节点： * *
- `nodeLLMSendResultsMultipleChoices` ：同时处理多个LLM选择
- `nodeSelectLLMChoice` ：将选择选择策略集成到工作流程中

* *增强控制流： * *
- 工具结果包含在列表中，以支持多种选择
- 在继续使用所选路径之前进行用户选择
- 所选选择被拆开并继续正常流程

优势 
- * *更好的控制* * ：与客服代表的工作流程进行精细整合
- * *灵活性* * ：可以与其他客服代表功能相结合
- * *透明度* * ：用户准确地看到人工智能正在考虑的内容

### 运行交互式 Agent

```
println("Chess Game started!")

val initialMessage = "Starting position is ${game.getBoard()}. White to move!"

runBlocking {
    agent.run(initialMessage)
}
```

```
Chess Game started!

Available LLM choices
Choice number 1: [Call(id=call_K46Upz7XoBIG5RchDh7bZE8F, tool=move, content={"notation": "p-e2-e4"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:17:40.368252Z, totalTokensCount=773, inputTokensCount=315, outputTokensCount=458, additionalInfo={}))]
Choice number 2: [Call(id=call_zJ6OhoCHrVHUNnKaxZkOhwoU, tool=move, content={"notation": "p-e2-e4"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:17:40.368252Z, totalTokensCount=773, inputTokensCount=315, outputTokensCount=458, additionalInfo={}))]
Choice number 3: [Call(id=call_nwX6ZMJ3F5AxiNUypYlI4BH4, tool=move, content={"notation": "p-e2-e4"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:17:40.368252Z, totalTokensCount=773, inputTokensCount=315, outputTokensCount=458, additionalInfo={}))]
Please choose a choice. Enter a number between 1 and 3: 
8 r n b q k b n r
7 p p p p p p p p
6 * * * * * * * *
5 * * * * * * * *
4 * * * * P * * *
3 * * * * * * * *
2 P P P P * P P P
1 R N B Q K B N R
  a b c d e f g h
- ----------------

Available LLM choices
Choice number 1: [Call(id=call_2V93GXOcIe0fAjUAIFEk9h5S, tool=move, content={"notation": "p-e7-e5"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:17:47.949303Z, totalTokensCount=1301, inputTokensCount=341, outputTokensCount=960, additionalInfo={}))]
Choice number 2: [Call(id=call_INM59xRzKMFC1w8UAV74l9e1, tool=move, content={"notation": "p-e7-e5"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:17:47.949303Z, totalTokensCount=1301, inputTokensCount=341, outputTokensCount=960, additionalInfo={}))]
Choice number 3: [Call(id=call_r4QoiTwn0F3jizepHH5ia8BU, tool=move, content={"notation": "p-e7-e5"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:17:47.949303Z, totalTokensCount=1301, inputTokensCount=341, outputTokensCount=960, additionalInfo={}))]
Please choose a choice. Enter a number between 1 and 3: 
8 r n b q k b n r
7 p p p p * p p p
6 * * * * * * * *
5 * * * * p * * *
4 * * * * P * * *
3 * * * * * * * *
2 P P P P * P P P
1 R N B Q K B N R
  a b c d e f g h
- ----------------

Available LLM choices
Choice number 1: [Call(id=call_f9XTizn41svcrtvnmkCfpSUQ, tool=move, content={"notation": "n-g1-f3"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:17:55.467712Z, totalTokensCount=917, inputTokensCount=341, outputTokensCount=576, additionalInfo={}))]
Choice number 2: [Call(id=call_c0Dfce5RcSbN3cOOm5ESYriK, tool=move, content={"notation": "n-g1-f3"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:17:55.467712Z, totalTokensCount=917, inputTokensCount=341, outputTokensCount=576, additionalInfo={}))]
Choice number 3: [Call(id=call_Lr4Mdro1iolh0fDyAwZsutrW, tool=move, content={"notation": "n-g1-f3"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:17:55.467712Z, totalTokensCount=917, inputTokensCount=341, outputTokensCount=576, additionalInfo={}))]
Please choose a choice. Enter a number between 1 and 3: 
8 r n b q k b n r
7 p p p p * p p p
6 * * * * * * * *
5 * * * * p * * *
4 * * * * P * * *
3 * * * * * N * *
2 P P P P * P P P
1 R N B Q K B * R
  a b c d e f g h
- ----------------

The execution was interrupted
```

```
import ai.koog.agents.core.feature.choice.nodeLLMSendResultsMultipleChoices
import ai.koog.agents.core.feature.choice.nodeSelectLLMChoice

inline fun <reified T> AIAgentSubgraphBuilderBase<*, *>.nodeTrimHistory(
    name: String? = null
): AIAgentNodeDelegate<T, T> = node(name) { result ->
    llm.writeSession {
        rewritePrompt { prompt ->
            val messages = prompt.messages

            prompt.copy(messages = listOf(messages.first(), messages.last()))
        }
    }

    result
}

val strategy = strategy<String, String>("chess_strategy") {
    val nodeCallLLM by nodeLLMRequest("sendInput")
    val nodeExecuteTool by nodeExecuteTool("nodeExecuteTool")
    val nodeSendToolResult by nodeLLMSendResultsMultipleChoices("nodeSendToolResult")
    val nodeSelectLLMChoice by nodeSelectLLMChoice(askChoiceStrategy, "chooseLLMChoice")
    val nodeTrimHistory by nodeTrimHistory<ReceivedToolResult>()

    edge(nodeStart forwardTo nodeCallLLM)
    edge(nodeCallLLM forwardTo nodeExecuteTool onToolCall { true })
    edge(nodeCallLLM forwardTo nodeFinish onAssistantMessage { true })
    edge(nodeExecuteTool forwardTo nodeTrimHistory)
    edge(nodeTrimHistory forwardTo nodeSendToolResult transformed { listOf(it) })
    edge(nodeSendToolResult forwardTo nodeSelectLLMChoice)
    edge(nodeSelectLLMChoice forwardTo nodeFinish transformed { it.first() } onAssistantMessage { true })
    edge(nodeSelectLLMChoice forwardTo nodeExecuteTool transformed { it.first() } onToolCall { true })
}
```

```
val game = ChessGame()
val toolRegistry = ToolRegistry { tools(listOf(Move(game))) }

val agent = AIAgent(
    executor = baseExecutor,
    strategy = strategy,
    llmModel = OpenAIModels.Chat.O3Mini,
    systemPrompt = """
            You are an agent who plays chess.
            You should always propose a move in response to the "Your move!" message.

            DO NOT HALLUCINATE!!!
            DO NOT PLAY ILLEGAL MOVES!!!
            YOU CAN SEND A MESSAGE ONLY IF IT IS A RESIGNATION OR A CHECKMATE!!!
        """.trimMargin(),
    temperature = 1.0,
    toolRegistry = toolRegistry,
    maxIterations = 200,
    numberOfChoices = 3,
)
```

```
println("Chess Game started!")

val initialMessage = "Starting position is ${game.getBoard()}. White to move!"

runBlocking {
    agent.run(initialMessage)
}
```

```
Chess Game started!
8 r n b q k b n r
7 p p p p p p p p
6 * * * * * * * *
5 * * * * * * * *
4 * * * * P * * *
3 * * * * * * * *
2 P P P P * P P P
1 R N B Q K B N R
  a b c d e f g h
- ----------------

Available LLM choices
Choice number 1: [Call(id=call_gqMIar0z11CyUl5nup3zbutj, tool=move, content={"notation": "p-e7-e5"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:18:17.313548Z, totalTokensCount=917, inputTokensCount=341, outputTokensCount=576, additionalInfo={}))]
Choice number 2: [Call(id=call_6niUGnZPPJILRFODIlJsCKax, tool=move, content={"notation": "p-e7-e5"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:18:17.313548Z, totalTokensCount=917, inputTokensCount=341, outputTokensCount=576, additionalInfo={}))]
Choice number 3: [Call(id=call_q1b8ZmIBph0EoVaU3Ic9A09j, tool=move, content={"notation": "p-e7-e5"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:18:17.313548Z, totalTokensCount=917, inputTokensCount=341, outputTokensCount=576, additionalInfo={}))]
Please choose a choice. Enter a number between 1 and 3: 
8 r n b q k b n r
7 p p p p * p p p
6 * * * * * * * *
5 * * * * p * * *
4 * * * * P * * *
3 * * * * * * * *
2 P P P P * P P P
1 R N B Q K B N R
  a b c d e f g h
- ----------------

Available LLM choices
Choice number 1: [Call(id=call_pdBIX7MVi82MyWwawTm1Q2ef, tool=move, content={"notation": "n-g1-f3"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:18:24.505344Z, totalTokensCount=1237, inputTokensCount=341, outputTokensCount=896, additionalInfo={}))]
Choice number 2: [Call(id=call_oygsPHaiAW5OM6pxhXhtazgp, tool=move, content={"notation": "n-g1-f3"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:18:24.505344Z, totalTokensCount=1237, inputTokensCount=341, outputTokensCount=896, additionalInfo={}))]
Choice number 3: [Call(id=call_GJTEsZ8J8cqOKZW4Tx54RqCh, tool=move, content={"notation": "n-g1-f3"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:18:24.505344Z, totalTokensCount=1237, inputTokensCount=341, outputTokensCount=896, additionalInfo={}))]
Please choose a choice. Enter a number between 1 and 3: 
8 r n b q k b n r
7 p p p p * p p p
6 * * * * * * * *
5 * * * * p * * *
4 * * * * P * * *
3 * * * * * N * *
2 P P P P * P P P
1 R N B Q K B * R
  a b c d e f g h
- ----------------

Available LLM choices
Choice number 1: [Call(id=call_5C7HdlTU4n3KdXcyNogE4rGb, tool=move, content={"notation": "n-g8-f6"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:18:34.646667Z, totalTokensCount=1621, inputTokensCount=341, outputTokensCount=1280, additionalInfo={}))]
Choice number 2: [Call(id=call_EjCcyeMLQ88wMa5yh3vmeJ2w, tool=move, content={"notation": "n-g8-f6"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:18:34.646667Z, totalTokensCount=1621, inputTokensCount=341, outputTokensCount=1280, additionalInfo={}))]
Choice number 3: [Call(id=call_NBMMSwmFIa8M6zvfbPw85NKh, tool=move, content={"notation": "n-g8-f6"}, metaInfo=ResponseMetaInfo(timestamp=2025-08-18T21:18:34.646667Z, totalTokensCount=1621, inputTokensCount=341, outputTokensCount=1280, additionalInfo={}))]
Please choose a choice. Enter a number between 1 and 3: 
8 r n b q k b * r
7 p p p p * p p p
6 * * * * * n * *
5 * * * * p * * *
4 * * * * P * * *
3 * * * * * N * *
2 P P P P * P P P
1 R N B Q K B * R
  a b c d e f g h
- ----------------

The execution was interrupted
```

交互式示例展示了用户如何指导人工智能的决策过程。在输出中，您可以看到：

1. * *多重选择* * ：人工智能生成3种不同的移动选项
2. * *用户选择* * ：用户输入数字1-3以选择他们的首选移动
3. * *游戏继续* * ：所选移动被执行，游戏继续

## 结论

本教程演示了使用Koog框架构建智能代理的几个关键方面：

### 关键收获

1. * *领域建模* * ：结构良好的数据模型对于复杂应用至关重要
2. * *工具集成* * ：自定义工具使代理能够有效地与外部系统交互
3. * *内存管理* * ：战略历史记录修剪可优化长时间交互的性能
4. * *策略图* * ： Koog基于图的方法提供了灵活的控制流程
5. * *交互式AI * * ：选择选择可实现人类AI协作和透明度

### 已探索的框架功能

- ✅ 自定义工具创建和集成
- ✅ 代理策略设计和基于图形的控制流程
- ✅ 内存优化技术
- ✅ 交互式选择选择
- ✅ 多个LLM回复处理
- ✅ 状态化游戏管理

Koog框架为构建复杂的人工智能代理奠定了基础，这些代理可以处理复杂的多回合交互，同时保持效率和透明度。
