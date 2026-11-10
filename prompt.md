# Dana's Bakery Order System — Codex Prompt

Build a Google Colab notebook that implements Dana's Bakery order system as a deterministic LangGraph workflow. Treat this file as the implementation specification. Do not call an LLM or any external API. The notebook must run in a fresh Colab session without an API key.

## Deliverables

Create one notebook named `dana_bakery_langgraph.ipynb`. It must:
- Install/import only the Python packages needed to run LangGraph and draw its graph in Colab.
- Define the graph state, node functions, routing functions, and compiled graph.
- Draw the compiled graph using `draw_mermaid_png()` and display the image in the notebook.
- Invoke the graph on at least six test orders of your own. Include tests that exercise every graph path, including invalid input, successful original item, successful substitution, failed substitutions/cancellation, Jerusalem delivery, out-of-Jerusalem pickup, discount, and Friday without discount. Print each result and assert that every result contains all six required output keys.
- Include short Markdown explanations of the rules, graph design, assumptions, and test coverage.

## Input contract

The graph must accept an initial state in this form:

```python
graph.invoke({
    "item": "babka",
    "quantity": 2,
    "day": "Thursday",
    "city": "Jerusalem",
})
```

The input fields are `item`, `quantity`, `day`, and `city`. Normalize item and city strings by trimming whitespace and comparing case-insensitively. The supported canonical products are `challah`, `babka`, `rugelach`, and `sourdough`. Do not accept croissants: they appear only on an obsolete menu. Treat an empty or whitespace-only city as pickup. Compare `Jerusalem` case-insensitively after trimming.

## Business rules

### Menu and prices

| Product | Unit price (ILS) | Opening stock |
|---|---:|---:|
| challah | 18 | 10 |
| babka | 45 | 0 |
| rugelach | 30 | 3 |
| sourdough | 25 | 4 |

Use the fixed opening stock above for each independent graph invocation. Do not decrement stock between separate invocations; no inventory persistence is required.

### Validation

- Reject an unknown product, including croissants, with a polite message.
- Reject a missing, non-integer, zero, or negative quantity with a polite message. Do not silently truncate a fractional quantity.
- A rejected order has `status="rejected"`, `item=None`, `total=0`, and a complete, polite `message`. Its `delivery` is still determined from the city rule so the output shape stays stable.

### Stock and substitutions

- A requested product is available only if stock is at least the full requested quantity.
- If the requested quantity is unavailable, try substitutes in this exact order:
  - babka → rugelach → sourdough → challah
  - rugelach → sourdough → challah
  - sourdough → challah
  - challah → no substitutes
- A substitute is valid only if its stock covers the full requested quantity. Keep the requested quantity unchanged and use the substitute's price.
- Make at most two substitute suggestions/attempts. If neither attempt can fulfill the full quantity, reject and apologize. Do not proceed to a third substitute. If the chain ends before two candidates exist, reject when no candidate remains.
- If the original product is available, sell it without substitution.

### Delivery

- An empty city means customer pickup, free of charge.
- Jerusalem orders use delivery and incur a 20 ILS delivery fee.
- Any non-empty city other than Jerusalem switches to free pickup. Tell the customer politely that delivery is unavailable outside Jerusalem and the order has been switched to pickup.

### Discount and total

- Apply Avi's final rule: give 15% off the baked-goods subtotal only when that subtotal is strictly greater than 150 ILS and the order day is not Friday.
- Friday orders never receive the discount. Compare the day case-insensitively after trimming.
- Calculate the discount before adding the delivery fee. Total is the discounted baked-goods subtotal plus any delivery fee.
- Keep monetary values numeric and round final amounts to two decimal places if needed.

### Customer reply

Generate replies deterministically in Python; do not use AI or an LLM. Keep the message polite, warm, and concise. Explain acceptance or cancellation, mention a substitution when one occurred, disclose the pickup switch for non-Jerusalem addresses, and state the final amount for accepted orders. Do not mention irrelevant story details such as the cat, the partner eating the babka, or the abandoned Instagram idea.

## LangGraph design

Use `StateGraph` with a typed state. Implement single-purpose nodes and conditional routing. The graph must include these logical stages (names may vary if the notebook clearly documents the mapping):

1. `validate_order`
2. `check_inventory`
3. `try_next_substitute` (repeat via conditional routing, with a state field tracking attempts)
4. `set_delivery`
5. `calculate_total`
6. `write_customer_message`
7. `reject_order`

Route invalid input directly to rejection. For a valid order, check stock; route an available item to delivery handling; route unavailable items to a substitute attempt while fewer than two attempts have been made and candidates remain; otherwise route to rejection. Accepted orders must pass through delivery handling, pricing, and message generation before END. Rejected orders must reach END with all required output fields populated.

Every node that executes must append its own node name to `steps` in execution order. Use a LangGraph reducer (for example `Annotated[list[str], operator.add]`) so updates append rather than replace the history. Avoid appending a node name twice for a single execution.

## Required result shape

Every `graph.invoke(...)` result, accepted or rejected, must contain these keys:

- `status`: exactly `"accepted"` or `"rejected"`
- `item`: canonical product actually sold, or `None` if rejected
- `delivery`: exactly `"pickup"` or `"delivery"`
- `total`: final amount to pay, including delivery, or 0 for a rejected order
- `message`: customer-facing reply
- `steps`: ordered list of executed graph node names

## Tests in the notebook

Write at least six meaningful test orders and use assertions, not just printed examples. Collectively cover:
- Invalid product rejection
- Invalid quantity rejection
- Direct sale with Jerusalem delivery
- At least one successful substitute
- Failed substitution path ending in cancellation
- Out-of-Jerusalem pickup switch
- Discount above the threshold on a non-Friday
- No discount on Friday
- All six required keys on every result
- Correct status, final item, delivery mode, total, and relevant node sequence for each case

It is acceptable for a single order to cover more than one case. Add a brief note describing which test covers each branch.

## Quality requirements

- Keep the notebook cells organized: setup, rules/state, nodes and routing, graph compilation and drawing, then test cases.
- Make the graph drawing readable and ensure the conditional branches are visible.
- Use explicit, understandable Python names and comments for business rules.
- Do not add unrelated features, web access, Instagram posting, persistent storage, or LLM behavior.
- Ensure the notebook can be run from top to bottom in a fresh Colab runtime.

## Prompt refinement log

The prompt was refined after reviewing the stock limits and the graph branches. The test quantities were adjusted so each expected outcome is possible with the stated opening inventory: a two-unit babka order reaches rugelach, a four-unit babka order reaches sourdough after rugelach is insufficient, and an eleven-unit babka order exercises cancellation after two failed substitute attempts. Additional cases were added for invalid quantities, the end of a shorter substitute chain, Jerusalem delivery, pickup outside Jerusalem, the greater-than-150 ILS discount, and the Friday exception. The final specification also makes the fixed per-invocation inventory, rejected-order output values, and ordered `steps` history explicit.
