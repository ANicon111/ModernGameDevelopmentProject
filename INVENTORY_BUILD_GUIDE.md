# Inventory System — Blueprint Build Guide (UE 5.8, BuildItUp)

Pure Blueprint. No C++. Scope: (1) core slots + stacking, (2) item data table, (3) UMG drag-and-drop grid.

Build the parts in this exact order — each part depends on the one before it. Inside Unreal, right-click in the Content Browser to create assets. Suggested folder: `Content/Inventory/`.

---

## Part 0 — Types (enum + struct)

### 0.1 Item category enum
1. Content Browser: right-click → Blueprints → **Enumeration**. Name it `E_ItemType`.
2. Add enumerators: `Potion`, `Tool`, `Ingredient`. (Coins are currency, not an item — stored separately, see 2.2.)

### 0.2 Item row struct
1. Right-click → Blueprints → **Structure**. Name it `S_ItemData`.
2. Add these variables (click **+ Add Variable** for each), with types:
   - `DisplayName` — **Name**
   - `Description` — **Text**
   - `Icon` — **Texture 2D** (Object Reference)
   - `MaxStack` — **Integer** (set default = `99`)
   - `ItemType` — `E_ItemType`
   - `Mesh` — **Static Mesh** (Object Reference) — optional, for dropping/placing items later
3. Compile/save.

### 0.3 Inventory slot struct
1. Right-click → Blueprints → **Structure**. Name it `S_InventorySlot`.
2. Add variables:
   - `ItemID` — **Name** (the DataTable row name; `None` = empty slot)
   - `Quantity` — **Integer** (default `0`)
3. Compile/save.

---

## Part 1 — Item data table (scope 2)

1. Right-click → Miscellaneous → **Data Table**.
2. In the pop-up, pick row structure `S_ItemData`. Name the asset `DT_Items`.
3. Open it. Click **+ Add** to add a row per item. Row Name = the item's ID. Fill the columns.
4. Only two item kinds: **magical potions** (`ItemType = Potion`, stackable) and **gathering tools** (`ItemType = Tool`, `MaxStack = 1`, not stackable). Suggested starter rows:

| Row Name | DisplayName | ItemType | MaxStack |
|---|---|---|---|
| HealthPotion | Health Potion | Potion | 10 |
| ManaPotion | Mana Potion | Potion | 10 |
| HerbSickle | Herb Sickle | Tool | 1 |
| EssenceFlask | Essence Flask | Tool | 1 |
| Herb | Herb | Ingredient | 99 |
| CrystalDust | Crystal Dust | Ingredient | 99 |
| Water | Spring Water | Ingredient | 99 |

Give each an `Icon` texture. Flow: tools gather **ingredients**; ingredients craft **potions**; potions and ingredients buy/sell for **coins**.

### 1.1 Buy/sell prices (for trading)
Add two more columns to `S_ItemData` (go back to the struct, add variables), so each item carries its own price:
- `BuyPrice` — **Integer** (coins to buy one)
- `SellPrice` — **Integer** (coins you get selling one)

Fill them on each `DT_Items` row (e.g. HealthPotion Buy 50 / Sell 20, Herb Buy 5 / Sell 2).

**Rule:** row name here IS the `ItemID` you store in slots. Keep them unique.

---

## Part 2 — Inventory component (scope 1: core + stacking)

This holds the data and all add/remove logic. An Actor Component so it lives on the character.

### 2.1 Create component
1. Right-click → Blueprint Class → expand **All Classes** → search `Actor Component` → select. Name it `AC_Inventory`.

### 2.2 Variables
Open `AC_Inventory`, add:
- `Slots` — **Array** of `S_InventorySlot`. This is the inventory.
- `MaxSlots` — **Integer**, default `4`. Instance Editable = on.
- `ItemDataTable` — **Data Table** reference. Instance Editable = on. (You'll set it to `DT_Items` on the character later, OR hard-set the default here to `DT_Items`.)
- `Coins` — **Integer**, default `0`. The player's currency. Instance Editable = on.

### 2.3 Event: on begin play — size the array
1. In the component's Event Graph, from **Event BeginPlay**:
2. Add a **For Loop** (First Index `0`, Last Index = `MaxSlots - 1`).
3. Loop Body → **Add** (to `Slots` array) a default `S_InventorySlot` (ItemID `None`, Quantity `0`).
   - This pre-fills fixed slots. (Skip this if you prefer a growing array — then AddItem just appends.)

### 2.4 Helper function: `GetItemData`
Input: `ItemID` (Name). Output: `S_ItemData` (as `Found Row`) + `Success` (Bool).
1. New Function `GetItemData`.
2. Node: **Get Data Table Row** — Data Table = `ItemDataTable`, Row Name = `ItemID` input.
3. Wire the `Out Row` to the output struct; wire exec `Row Found` → return Success = true, `Row Not Found` → Success = false.

### 2.5 Function: `AddItem`
Inputs: `ItemID` (Name), `Amount` (Integer, default 1). Output: `Remainder` (Integer) — what didn't fit.

Logic (order matters):
1. Call `GetItemData(ItemID)`. If not Success → return (invalid item). Read `MaxStack`.
2. **First pass — top up existing stacks.** For Each (index) in `Slots`:
   - If slot `ItemID == ItemID` AND `Quantity < MaxStack`:
     - `Space = MaxStack - Quantity`
     - `ToAdd = min(Space, Amount)`
     - Set slot.Quantity += ToAdd; `Amount -= ToAdd`.
     - If `Amount <= 0` → break, go to step 4.
3. **Second pass — fill empty slots.** For Each (index) in `Slots`:
   - If slot `ItemID == None` (empty):
     - `ToAdd = min(MaxStack, Amount)`
     - Set slot.ItemID = ItemID, slot.Quantity = ToAdd; `Amount -= ToAdd`.
     - If `Amount <= 0` → break.
4. Set `Remainder = Amount`.
5. **Call the update dispatcher** (see 2.7) so UI refreshes.

> Writing array-of-struct members: use **Set Members in S_InventorySlot** on the array element, then write it back with the array index, OR right-click the array pin → it's cleaner to do `Set Array Elem`. Simplest reliable pattern: get the struct by index → **Set members** → **Set** array element at that index.

### 2.6 Function: `RemoveItem`
Inputs: `SlotIndex` (Integer), `Amount` (Integer, default 1).
1. Get `Slots[SlotIndex]`.
2. `Quantity -= Amount`. If `Quantity <= 0` → reset slot to empty (ItemID `None`, Quantity `0`).
3. Write back. Call update dispatcher.

### 2.6b Coin + trade functions
- `AddCoins` — input `Amount` (Integer). `Coins += Amount`. Call `OnInventoryUpdated`.
- `SpendCoins` — input `Amount` (Integer), output `Success` (Bool). If `Coins >= Amount` → `Coins -= Amount`, Success = true, call `OnInventoryUpdated`. Else Success = false.
- `BuyItem` — input `ItemID` (Name), `Amount` (Integer, default 1).
  1. `GetItemData(ItemID)` → read `BuyPrice`. `Cost = BuyPrice * Amount`.
  2. Call `SpendCoins(Cost)`. If false → return (not enough coins).
  3. Call `AddItem(ItemID, Amount)`.
- `SellItem` — input `SlotIndex` (Integer), `Amount` (Integer, default 1).
  1. Read `Slots[SlotIndex].ItemID` → `GetItemData` → read `SellPrice`. Clamp `Amount` to the slot's `Quantity`.
  2. `AddCoins(SellPrice * Amount)`.
  3. `RemoveItem(SlotIndex, Amount)`.

> `BuyItem`/`SellItem` are the trade logic. Hook a shop widget's buttons to them later. Crafting potions from ingredients = a later `CraftItem` function (recipe check: has ingredients → RemoveItem each → AddItem potion); not in this scope.

### 2.7 Event Dispatcher: `OnInventoryUpdated`
1. In `AC_Inventory` → **Event Dispatchers** panel → **+**. Name `OnInventoryUpdated`.
2. In both `AddItem` and `RemoveItem`, at the end, **Call** `OnInventoryUpdated`.
   - This is how the UI knows to redraw. UI binds to it in Part 3.

Compile/save.

---

## Part 3 — UMG drag-and-drop UI (scope 3)

Two widgets: one slot, one grid of slots. Plus a drag payload object.

### 3.1 Drag payload
1. Blueprint Class → All Classes → search `DragDropOperation`... actually create: Blueprint Class → **Object** → name `DragItemPayload`.
2. Add variables: `FromSlotIndex` (Integer), `OwningInventory` (`AC_Inventory` reference).

### 3.2 Slot widget — `WBP_InventorySlot`
1. Right-click → User Interface → **Widget Blueprint** → Parent = User Widget. Name `WBP_InventorySlot`.
2. Designer layout:
   - Root: **Size Box** (Width/Height e.g. 64×64) → **Border** (for background) → **Overlay**:
     - **Image** named `Img_Icon` (the item icon).
     - **Text** named `Txt_Quantity`, bottom-right aligned (stack count).
3. Variables on the widget:
   - `SlotIndex` (Integer), Instance Editable + **Expose on Spawn** = on.
   - `OwningInventory` (`AC_Inventory` ref), Expose on Spawn = on.
4. Function `Refresh`:
   - Read `OwningInventory.Slots[SlotIndex]`.
   - If `ItemID == None` → hide `Img_Icon`, clear `Txt_Quantity`.
   - Else → `GetItemData(ItemID)` → set `Img_Icon` Brush from `Icon` texture; set `Txt_Quantity` = `Quantity` (hide text if Quantity <= 1).
5. Call `Refresh` on **Event Construct**.

### 3.3 Drag-and-drop on the slot
In `WBP_InventorySlot` Graph:
1. Override **On Mouse Button Down** → return **Detect Drag If Pressed** (Left Mouse Button). (Right-click graph → add `Detect Drag if Pressed`, wire its drag key.)
2. Override **On Drag Detected**:
   - Create `DragItemPayload`, set `FromSlotIndex = SlotIndex`, `OwningInventory = OwningInventory`.
   - **Create Drag Drop Operation**, set its `Payload` = the payload object, `Default Drag Visual` = a new `WBP_InventorySlot` (or an Image of the icon), `Pivot` = Mouse Down.
   - Return that operation.
3. Override **On Drop**:
   - Cast dropped `Operation.Payload` to `DragItemPayload`.
   - `From = Payload.FromSlotIndex`, `To = this SlotIndex`.
   - **Swap** the two slots in `OwningInventory.Slots` (get both, set each to the other). Then call `OwningInventory.OnInventoryUpdated` (so whole grid refreshes).
   - Return true.

### 3.4 Grid widget — `WBP_Inventory`
1. Widget Blueprint named `WBP_Inventory`.
2. Designer: a **Wrap Box** or **Uniform Grid Panel** named `SlotContainer`, inside a centered Border/background panel. Add a **Text** named `Txt_Coins` at the top for the coin count.
3. Variable `OwningInventory` (`AC_Inventory` ref), Expose on Spawn = on.
4. Function `BuildGrid`:
   - **Clear Children** of `SlotContainer`.
   - For Each index in `OwningInventory.Slots`:
     - **Create Widget** `WBP_InventorySlot`, set `SlotIndex = index`, `OwningInventory = OwningInventory`.
     - **Add Child** to `SlotContainer`.
5. Function `RefreshCoins`: set `Txt_Coins` = `"Coins: " + OwningInventory.Coins`.
6. **Event Construct**: call `BuildGrid` + `RefreshCoins`, then **Bind** to `OwningInventory.OnInventoryUpdated` → on fire, call both `BuildGrid` and `RefreshCoins` (full rebuild — simple and correct; optimize later).

---

## Part 4 — Wire to character + input

### 4.1 Add component to character
1. Open `BP_ThirdPersonCharacter`.
2. Components panel → **+ Add** → `AC_Inventory`.
3. Select it → in Details set `ItemDataTable = DT_Items` (if not already defaulted) and `MaxSlots` as wanted.

### 4.2 Toggle input
1. Content Browser `Content/Input/Actions`: right-click → Input → **Input Action**. Name `IA_ToggleInventory`. Value Type = Digital (bool).
2. Open `Content/Input/IMC_Default`. Add a mapping: `IA_ToggleInventory` → key **I** (or **Tab**).

### 4.3 Open/close logic (in BP_ThirdPersonCharacter)
Variables: `InventoryWidgetRef` (`WBP_Inventory` ref), `bInventoryOpen` (Bool).
1. Event graph → **EnhancedInputAction IA_ToggleInventory** (Triggered).
2. **FlipFlop**:
   - **A (open):**
     - Create Widget `WBP_Inventory`, set `OwningInventory` = this character's `AC_Inventory`. Store in `InventoryWidgetRef`.
     - **Add to Viewport**.
     - Get Player Controller → **Set Input Mode Game And UI** (widget to focus = the inventory) + **Set Show Mouse Cursor = true**.
   - **B (close):**
     - `InventoryWidgetRef` → **Remove from Parent**.
     - Set Input Mode Game Only + Show Mouse Cursor = false.

### 4.4 Test giving items
Quick test: on `BeginPlay` of the character (or a key press), call `AC_Inventory → AddItem` with `ItemID = HealthPotion`, `Amount = 5`. Open inventory (press I) → see 5 HealthPotion stack. Add `HerbSickle` Amount 2 → two separate slots (MaxStack 1, tools never stack).

---

## Part 5 — Brewing / crafting potions (core)

Turn ingredients into potions via recipes. Data-driven so you add recipes without code.

### 5.1 Recipe struct
Right-click → Blueprints → **Structure**. Name `S_Recipe`. Variables:
- `Ingredient1` — **Name** (DT_Items row, an Ingredient)
- `Amount1` — **Integer**
- `Ingredient2` — **Name** (set `None` if recipe uses one ingredient)
- `Amount2` — **Integer**
- `Ingredient3` — **Name** (`None` if unused)
- `Amount3` — **Integer**
- `ResultItem` — **Name** (the potion/artifact produced)
- `ResultAmount` — **Integer** (default `1`)

> Fixed 3 ingredient slots = simple + covers GDD "different combinations." Want unlimited? Use an array of a small `S_RecipeEntry {ItemID, Amount}` struct instead; same logic, harder nodes.

### 5.2 Recipe table
Right-click → **Data** (or Miscellaneous) → **Data Table**. Row structure `S_Recipe`. Name `DT_Recipes`. Add rows, e.g.:

| Row Name | Ingredient1/Amount1 | Ingredient2/Amount2 | ResultItem/ResultAmount |
|---|---|---|---|
| HealthPotion | Herb / 2 | Water / 1 | HealthPotion / 1 |
| ManaPotion | CrystalDust / 2 | Water / 1 | ManaPotion / 1 |

Row Name = recipe ID (can match the potion). Leave `Ingredient3` = `None` when unused.

### 5.3 Count helper on `AC_Inventory`
Function `GetItemCount` — input `ItemID` (Name), output `Total` (Integer).
- For Each slot in `Slots`: if `slot.ItemID == ItemID`, add `slot.Quantity` to a local Total. Return Total.

### 5.4 `RemoveItemByID` on `AC_Inventory`
Needed because brewing removes by item type, not a known slot index.
Input `ItemID` (Name), `Amount` (Integer), output `Success` (Bool).
1. If `GetItemCount(ItemID) < Amount` → Success = false, return.
2. For Each slot: if `slot.ItemID == ItemID`:
   - `Take = min(slot.Quantity, Amount)`; `slot.Quantity -= Take`; `Amount -= Take`.
   - If `slot.Quantity <= 0` → reset slot empty.
   - If `Amount <= 0` → break.
3. Success = true. Call `OnInventoryUpdated`.

### 5.5 `CanCraft` on `AC_Inventory`
Input `RecipeID` (Name), output `bool`.
1. **Get Data Table Row** from `DT_Recipes` by `RecipeID`. Not found → false.
2. For each non-`None` ingredient (1–3): check `GetItemCount(IngredientN) >= AmountN`. Any fail → false.
3. All pass → true.

### 5.6 `CraftItem` on `AC_Inventory`
Input `RecipeID` (Name), output `Success` (Bool).
1. Call `CanCraft(RecipeID)`. False → Success = false, return.
2. Read recipe row. For each non-`None` ingredient: `RemoveItemByID(IngredientN, AmountN)`.
3. `AddItem(ResultItem, ResultAmount)`.
4. Success = true. (`AddItem`/`RemoveItemByID` already fire `OnInventoryUpdated`.)

### 5.7 Brewing UI — `WBP_PotionBook`
1. Widget Blueprint `WBP_PotionBook`. Variable `OwningInventory` (`AC_Inventory` ref), Expose on Spawn.
2. Designer: a **Scroll Box** named `RecipeList`.
3. Make a small row widget `WBP_RecipeEntry`: Text `Txt_Name`, Text `Txt_Ingredients`, Button `Btn_Brew`. Vars `RecipeID` (Name), `OwningInventory`, both Expose on Spawn.
   - On Construct: read recipe row → set `Txt_Name` = ResultItem, `Txt_Ingredients` = list of "AmountN x IngredientN". Set `Btn_Brew` **IsEnabled** = `OwningInventory.CanCraft(RecipeID)`.
   - `Btn_Brew` OnClicked → `OwningInventory.CraftItem(RecipeID)` → then re-check enabled on all entries (simplest: rebuild list).
4. `WBP_PotionBook` Construct: For Each row in `DT_Recipes` (**Get Data Table Row Names**) → Create `WBP_RecipeEntry`, set RecipeID + OwningInventory → Add to `RecipeList`.
5. Open it with its own key or a cauldron interaction (same add-to-viewport pattern as Part 4.3).

> This is the recipe-book form of brewing. GDD also mentions a crafting-grid. Grid variant = place ingredients in grid cells, match the set against recipes; more UI work, same `CanCraft`/`CraftItem` core. Start with the book.

---

## Part 6 — Haggling sale (core)

Replaces fixed-price selling for customers. Customer has a hidden willingness-to-pay; player proposes a price; bluff can push it higher but risks losing the sale.

### 6.1 Customer data
On your customer actor (or a `S_Customer` struct), per sale:
- `WantedItem` — **Name** (potion the customer wants)
- `FairValue` — **Integer** = that item's `SellPrice` from `DT_Items`.
- `MaxPay` — **Integer** = `FairValue * RandomFloatInRange(1.0, 1.6)`, rounded. Hidden from player. Roll once when customer spawns.
- `Patience` — **Integer**, default `3` (counter-offers before they leave).

### 6.2 Haggle function (on customer, or a `BP_HaggleManager`)
Input `AskPrice` (Integer), `bBluff` (Bool). Outputs: `Result` (enum `E_HaggleResult`: `Accepted`, `Countered`, `Left`), `CounterOffer` (Integer).
1. Make enum `E_HaggleResult` with `Accepted`, `Countered`, `Left`.
2. `EffectiveMax = MaxPay`.
   - If `bBluff`: with some chance (e.g. `RandomFloat < 0.5`) bluff works → `EffectiveMax = MaxPay * 1.2`. Else bluff caught → `Patience -= 1` (extra penalty) and play heartbeat sound (GDD).
3. If `AskPrice <= EffectiveMax` → `Result = Accepted`.
4. Else: `Patience -= 1`. If `Patience <= 0` → `Result = Left`. Else `Result = Countered`, `CounterOffer = (AskPrice + EffectiveMax) / 2`.

### 6.3 On accepted sale
1. Confirm player has the item: `GetItemCount(WantedItem) >= 1`.
2. `RemoveItemByID(WantedItem, 1)`.
3. `AddCoins(AskPrice)`.

### 6.4 Haggle UI — `WBP_Haggle`
- Show customer's `WantedItem` + a price input (Slider or +/- Text).
- Buttons: **Offer** (bBluff=false) and **Bluff** (bBluff=true).
- On click → call haggle function → branch on `Result`:
  - `Accepted` → do 6.3, show "Sold!", close.
  - `Countered` → show `CounterOffer`, let player offer again.
  - `Left` → show "Customer left", close.
- Heartbeat sound: **Play Sound 2D** when bluff is caught (from 6.2).

> Keep `SellItem` (fixed price) for quick/non-haggle sales or testing. Haggle is the customer-facing path.

---

## Build order checklist
1. `E_ItemType`, `S_ItemData`, `S_InventorySlot`
2. `DT_Items` (+ test rows)
3. `AC_Inventory` (vars, BeginPlay sizing, GetItemData, AddItem, RemoveItem, OnInventoryUpdated)
4. `DragItemPayload`, `WBP_InventorySlot` (+ drag/drop), `WBP_Inventory`
5. Add component to `BP_ThirdPersonCharacter`; `IA_ToggleInventory` + `IMC_Default` mapping; toggle logic; test AddItem
6. Brewing: `S_Recipe`, `DT_Recipes`, `GetItemCount`, `RemoveItemByID`, `CanCraft`, `CraftItem`, `WBP_PotionBook` (+ `WBP_RecipeEntry`)
7. Haggle: `E_HaggleResult`, customer data, haggle function, accepted-sale logic, `WBP_Haggle`

Maps to the 3 core loops: **collect** (Part 2 AddItem via foraging pickup) → **brew** (Part 5) → **haggle/sell** (Part 6).

## Later (not in this scope)
- Shop customisation (buy furniture from catalog, place/decorate) — extra feature, skipped
- Crafting-grid variant of brewing (vs the recipe book)
- `Artifact` item type (GDD mentions artifacts alongside potions)
- Save/load (SaveGame serializing `Slots` + `Coins`)
- Right-click to use/drop, split stacks, tooltips, foraging pickup actor

---

---

## Appendix A — `AddItem` node map (simple, 4 slots)

Fast version. No max-stack cap, no break macros. Uses `Return Node` inside loops for early exit (legal in functions). Set `MaxSlots = 4`. Stacks infinitely.

Notation: `Node.Pin -> Node.Pin`. `exec` = white arrow.
Nodes: `AddItem`(entry), `GetItemData`, 3x `Return Node`, 1x `Branch`(valid), 2x `Branch`(match/empty), `Break S_ItemData` (only if you still need MaxStack elsewhere — else skip), 2x `Break S_InventorySlot`, 2x `Set members in S_InventorySlot`, 2x `Set Array Elem`, 2x `For Each Loop` (plain, NOT with-break), 2x `Equal (Name)`, 1x `+`(int).
Not needed anymore: `<`, `<=`, `AND`, `Min`, `-`, `MaxStackLocal`, `Remaining`. Delete them.

### Section 1 — validate
- `AddItem.exec -> GetItemData.exec`
- `AddItem.ItemID -> GetItemData.ItemID`
- `GetItemData.exec -> BranchValid.exec`
- `GetItemData.Success -> BranchValid.Condition`
- `BranchValid.False -> ReturnInvalid.exec`  (item not in DT_Items; nothing added)
- `BranchValid.True -> ForEach1.exec`

### Section 2 — Pass 1: stack existing (ForEach1, plain For Each Loop, Array = Slots)
- `ForEach1.Array Element -> BreakSlot1.(S Inventory Slot)`
- `ForEach1.Loop Body -> B1.exec`  (B1 = Branch)
- `eq1 = Equal(Name)`: `BreakSlot1.ItemID -> A`, `AddItem.ItemID -> B`
- `eq1 -> B1.Condition`
- `B1.True ->`:
  - `plus = (int +)`: `BreakSlot1.Quantity -> A`, `AddItem.Amount -> B`
  - `SetMembers1 = Set members in S_InventorySlot`: struct in = `ForEach1.Array Element`; tick **Quantity** only; `plus -> Quantity`
  - `SetArrayElem1 = Set Array Elem`: Array = `Slots`, Index = `ForEach1.Array Index`, Item = `SetMembers1.(out)`
  - exec: `B1.True -> SetMembers1 -> SetArrayElem1 -> Return1`  (early exit: item stacked)
- `B1.False -> (empty; loop continues)`
- `ForEach1.Completed -> ForEach2.exec`

### Section 3 — Pass 2: first empty slot (ForEach2, plain, Array = Slots)
- `ForEach2.Array Element -> BreakSlot2.(S Inventory Slot)`
- `ForEach2.Loop Body -> B2.exec`
- `eq2 = Equal(Name)`: `BreakSlot2.ItemID -> A`, B = leave blank (= `None`, empty slot)
- `eq2 -> B2.Condition`
- `B2.True ->`:
  - `SetMembers2 = Set members in S_InventorySlot`: struct in = `ForEach2.Array Element`; tick **Item ID** + **Quantity**; `AddItem.ItemID -> Item ID`, `AddItem.Amount -> Quantity`
  - `SetArrayElem2 = Set Array Elem`: Array = `Slots`, Index = `ForEach2.Array Index`, Item = `SetMembers2.(out)`
  - exec: `B2.True -> SetMembers2 -> SetArrayElem2 -> Return2`  (early exit: placed in empty slot)
- `B2.False -> (empty)`
- `ForEach2.Completed -> Return3`  (inventory full, 4 slots used, nothing added)

Later (after 2.7 dispatcher): call `OnInventoryUpdated` just before `Return1` and `Return2`.

> `Return Node` inside a loop exits the whole function immediately = clean break. Plain `For Each Loop` (not with-break) is enough.
> Writing array-of-struct needs `Set members` + `Set Array Elem`. Editing `Array Element` alone does NOT save.
> `RemoveItem` (2.6): `Set members`(Quantity -= Amount; if <=0 reset Item ID=None) then `Set Array Elem` at that index.
