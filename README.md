# NCCL Experiment Repository

## MANA: MPI-Agnostic, Network-Agnostic Transparent Checkpointing

<p>The checkpointing process in MANA follows a carefully orchestrated multi-phase approach to ensure application state consistency while maintaining transparency.</p>

![Checkpoint Process Flow](dot_inline_dotgraph_1.png)

## Process Flows

### Restart Process Flow

The restart process reconstructs the entire application state from checkpoint files while potentially using different MPI implementations or network fabrics.

![Restart Process Flow](dot_inline_dotgraph_2.png)

### MPI Call Interception Flow

Every MPI call goes through MANA's interception layer to provide transparency and enable checkpointing.

![MPI Call Flow](dot_inline_dotgraph_3.png)

### Virtual Object Management Flow

Virtual objects enable MANA's MPI/network agnosticism by abstracting implementation details.

![Virtual Object Flow](dot_inline_dotgraph_4.png)

## Detailed Documentation

For comprehensive technical details, architecture explanations, and implementation specifics, please refer to:

**[MANA_CODEBASE_SUMMARY.md](MANA_CODEBASE_SUMMARY.md)**
