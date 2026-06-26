# Main Battle Scene Demo Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a playable Godot 4.7 main battle scene demo with one player, one enemy, card play, energy, health, block, turn flow, readable dark fantasy UI, generated visual assets, and victory/defeat states.

**Architecture:** Keep battle rules in small plain GDScript state classes and keep scene nodes focused on rendering and input. `BattleController.gd` is the only object that mutates combat state during play; UI nodes call controller methods and re-render from state snapshots. The first version uses click-to-play and single-enemy auto-targeting to avoid unnecessary targeting complexity.

**Tech Stack:** Godot 4.7 stable, GDScript, built-in Godot UI nodes, generated PNG/WebP bitmap assets stored under `assets/generated/`.

## Global Constraints

- Build only the main battle scene demo; do not add map, rewards, relics, potions, shops, save systems, or run progression.
- Use one player and one enemy.
- Start with a 10-card deck: 4 Strike, 4 Defend, 1 Heavy Blow, 1 Quick Thought.
- Support exactly these card effect types for the first demo: `damage`, `block`, `draw`.
- Use click-to-play; drag-to-play is optional and should not block completion.
- Attacks target the single enemy automatically.
- Enemy intent must be visible before the enemy acts.
- Generated image assets must use model `gpt-image-2`; credentials must stay outside git.
- Store generated assets under `assets/generated/`.
- The demo must start directly in `scenes/BattleScene.tscn`.

---

## File Structure

- Create `scripts/cards/CardData.gd`: immutable card definition helper with `duplicate_instance()` for deck copies.
- Create `scripts/cards/CardLibrary.gd`: returns the fixed starter deck and card definitions.
- Create `scripts/battle/PlayerState.gd`: player health, block, energy, draw pile, hand, discard pile, draw and discard operations.
- Create `scripts/battle/EnemyState.gd`: enemy health, block, repeating intent pattern, damage/block operations.
- Create `scripts/battle/BattleController.gd`: battle state machine, play-card validation, card effect application, enemy turn, victory/defeat.
- Create `scripts/ui/CardView.gd`: one clickable card UI control.
- Create `scripts/ui/EnemyView.gd`: enemy display and intent display binding.
- Create `scripts/ui/FloatingText.gd`: short-lived combat feedback label.
- Create `scenes/CardView.tscn`: reusable card control scene.
- Create `scenes/EnemyView.tscn`: reusable enemy panel scene.
- Create `scenes/FloatingText.tscn`: reusable floating text scene.
- Create `scenes/BattleScene.tscn`: main scene with background, player panel, enemy view, hand container, piles, energy, end turn button, overlays.
- Modify `project.godot`: set `application/run/main_scene="res://scenes/BattleScene.tscn"` and keep existing Godot 4.7 settings.
- Create `assets/generated/.gitkeep`: reserves the generated asset directory without committing secrets.
- Create `tests/manual/battle-demo-checklist.md`: manual verification checklist for the playable demo.

---

### Task 1: Card And Combat State Core

**Files:**
- Create: `scripts/cards/CardData.gd`
- Create: `scripts/cards/CardLibrary.gd`
- Create: `scripts/battle/PlayerState.gd`
- Create: `scripts/battle/EnemyState.gd`
- Create: `scripts/battle/BattleController.gd`

**Interfaces:**
- Produces: `CardData.new(id: String, display_name: String, cost: int, card_type: String, description: String, effects: Array[Dictionary])`
- Produces: `CardLibrary.create_starter_deck() -> Array[CardData]`
- Produces: `PlayerState.setup(max_health: int, deck: Array[CardData]) -> void`
- Produces: `PlayerState.start_turn(energy_amount: int, draw_count: int) -> void`
- Produces: `EnemyState.setup(max_health: int, intents: Array[Dictionary]) -> void`
- Produces: `EnemyState.get_current_intent() -> Dictionary`
- Produces: `BattleController.setup_battle() -> void`
- Produces: `BattleController.play_card(card: CardData) -> bool`
- Produces: `BattleController.end_player_turn() -> void`
- Produces: `BattleController.get_state() -> Dictionary`

- [ ] **Step 1: Write `CardData.gd`**

```gdscript
extends RefCounted
class_name CardData

var id: String
var display_name: String
var cost: int
var card_type: String
var description: String
var effects: Array[Dictionary]

func _init(
        card_id: String = "",
        card_name: String = "",
        card_cost: int = 0,
        type_name: String = "",
        card_description: String = "",
        card_effects: Array[Dictionary] = []
) -> void:
    id = card_id
    display_name = card_name
    cost = card_cost
    card_type = type_name
    description = card_description
    effects = card_effects.duplicate(true)

func duplicate_instance() -> CardData:
    return CardData.new(id, display_name, cost, card_type, description, effects)
```

- [ ] **Step 2: Write `CardLibrary.gd`**

```gdscript
extends RefCounted
class_name CardLibrary

static func create_card(card_id: String) -> CardData:
    match card_id:
        "strike":
            return CardData.new("strike", "Strike", 1, "Attack", "Deal 6 damage.", [
                {"type": "damage", "amount": 6, "target": "enemy"},
            ])
        "defend":
            return CardData.new("defend", "Defend", 1, "Skill", "Gain 5 block.", [
                {"type": "block", "amount": 5, "target": "player"},
            ])
        "heavy_blow":
            return CardData.new("heavy_blow", "Heavy Blow", 2, "Attack", "Deal 12 damage.", [
                {"type": "damage", "amount": 12, "target": "enemy"},
            ])
        "quick_thought":
            return CardData.new("quick_thought", "Quick Thought", 0, "Skill", "Draw 1 card.", [
                {"type": "draw", "amount": 1, "target": "player"},
            ])
        _:
            push_warning("Unknown card id: %s" % card_id)
            return CardData.new("missing", "Missing Card", 0, "Skill", "No effect.", [])

static func create_starter_deck() -> Array[CardData]:
    var deck: Array[CardData] = []
    for index in range(4):
        deck.append(create_card("strike"))
        deck.append(create_card("defend"))
    deck.append(create_card("heavy_blow"))
    deck.append(create_card("quick_thought"))
    return deck
```

- [ ] **Step 3: Write `PlayerState.gd`**

```gdscript
extends RefCounted
class_name PlayerState

var max_health: int = 70
var health: int = 70
var block: int = 0
var energy: int = 0
var draw_pile: Array[CardData] = []
var hand: Array[CardData] = []
var discard_pile: Array[CardData] = []

func setup(starting_max_health: int, deck: Array[CardData]) -> void:
    max_health = starting_max_health
    health = starting_max_health
    block = 0
    energy = 0
    draw_pile.clear()
    hand.clear()
    discard_pile.clear()
    for card in deck:
        draw_pile.append(card.duplicate_instance())
    draw_pile.shuffle()

func start_turn(energy_amount: int, draw_count: int) -> void:
    block = 0
    energy = energy_amount
    draw_cards(draw_count)

func draw_cards(count: int) -> void:
    for index in range(count):
        if draw_pile.is_empty():
            _reshuffle_discard_into_draw()
        if draw_pile.is_empty():
            return
        hand.append(draw_pile.pop_back())

func spend_energy(amount: int) -> bool:
    if energy < amount:
        return false
    energy -= amount
    return true

func gain_block(amount: int) -> void:
    block += max(amount, 0)

func take_damage(amount: int) -> void:
    var incoming := max(amount, 0)
    var blocked := min(block, incoming)
    block -= blocked
    incoming -= blocked
    health = max(health - incoming, 0)

func discard_card(card: CardData) -> void:
    var index := hand.find(card)
    if index >= 0:
        hand.remove_at(index)
        discard_pile.append(card)

func discard_hand() -> void:
    while not hand.is_empty():
        discard_pile.append(hand.pop_back())

func _reshuffle_discard_into_draw() -> void:
    while not discard_pile.is_empty():
        draw_pile.append(discard_pile.pop_back())
    draw_pile.shuffle()
```

- [ ] **Step 4: Write `EnemyState.gd`**

```gdscript
extends RefCounted
class_name EnemyState

var max_health: int = 50
var health: int = 50
var block: int = 0
var intents: Array[Dictionary] = []
var intent_index: int = 0

func setup(starting_max_health: int, starting_intents: Array[Dictionary]) -> void:
    max_health = starting_max_health
    health = starting_max_health
    block = 0
    intents = starting_intents.duplicate(true)
    intent_index = 0

func get_current_intent() -> Dictionary:
    if intents.is_empty():
        return {"type": "attack", "amount": 0, "label": "Idle"}
    return intents[intent_index % intents.size()]

func advance_intent() -> void:
    if not intents.is_empty():
        intent_index = (intent_index + 1) % intents.size()

func gain_block(amount: int) -> void:
    block += max(amount, 0)

func take_damage(amount: int) -> void:
    var incoming := max(amount, 0)
    var blocked := min(block, incoming)
    block -= blocked
    incoming -= blocked
    health = max(health - incoming, 0)
```

- [ ] **Step 5: Write `BattleController.gd`**

```gdscript
extends Node
class_name BattleController

signal state_changed(state: Dictionary)
signal floating_text_requested(text: String, world_position: Vector2, color: Color)
signal battle_finished(result: String)
signal turn_banner_requested(text: String)

const STARTING_ENERGY := 3
const STARTING_HAND_SIZE := 5

var player := PlayerState.new()
var enemy := EnemyState.new()
var is_player_turn := false
var is_battle_over := false

func _ready() -> void:
    setup_battle()

func setup_battle() -> void:
    player.setup(70, CardLibrary.create_starter_deck())
    enemy.setup(50, [
        {"type": "attack", "amount": 6, "label": "Attack"},
        {"type": "attack", "amount": 9, "label": "Heavy Attack"},
        {"type": "block", "amount": 6, "label": "Guard"},
    ])
    is_battle_over = false
    _start_player_turn()

func play_card(card: CardData) -> bool:
    if is_battle_over or not is_player_turn:
        push_warning("Cannot play card outside player turn.")
        return false
    if not player.hand.has(card):
        push_warning("Cannot play card that is not in hand.")
        return false
    if player.energy < card.cost:
        push_warning("Not enough energy to play %s." % card.display_name)
        _emit_state()
        return false
    if not player.spend_energy(card.cost):
        return false

    for effect in card.effects:
        _apply_effect(effect)

    player.discard_card(card)
    _check_battle_end()
    _emit_state()
    return true

func end_player_turn() -> void:
    if is_battle_over or not is_player_turn:
        return
    is_player_turn = false
    player.discard_hand()
    turn_banner_requested.emit("Enemy Turn")
    _resolve_enemy_turn()

func get_state() -> Dictionary:
    return {
        "is_player_turn": is_player_turn,
        "is_battle_over": is_battle_over,
        "player_health": player.health,
        "player_max_health": player.max_health,
        "player_block": player.block,
        "player_energy": player.energy,
        "enemy_health": enemy.health,
        "enemy_max_health": enemy.max_health,
        "enemy_block": enemy.block,
        "enemy_intent": enemy.get_current_intent(),
        "hand": player.hand,
        "draw_count": player.draw_pile.size(),
        "discard_count": player.discard_pile.size(),
    }

func _start_player_turn() -> void:
    is_player_turn = true
    player.start_turn(STARTING_ENERGY, STARTING_HAND_SIZE)
    turn_banner_requested.emit("Player Turn")
    _emit_state()

func _resolve_enemy_turn() -> void:
    var intent := enemy.get_current_intent()
    match String(intent.get("type", "")):
        "attack":
            var amount := int(intent.get("amount", 0))
            player.take_damage(amount)
            floating_text_requested.emit("-%d" % amount, Vector2(320, 430), Color(1.0, 0.25, 0.2))
        "block":
            var amount := int(intent.get("amount", 0))
            enemy.gain_block(amount)
            floating_text_requested.emit("+%d Block" % amount, Vector2(900, 260), Color(0.45, 0.75, 1.0))
        _:
            push_warning("Unknown enemy intent: %s" % intent)

    enemy.advance_intent()
    _check_battle_end()
    if not is_battle_over:
        _start_player_turn()
    else:
        _emit_state()

func _apply_effect(effect: Dictionary) -> void:
    var effect_type := String(effect.get("type", ""))
    var amount := int(effect.get("amount", 0))
    match effect_type:
        "damage":
            enemy.take_damage(amount)
            floating_text_requested.emit("-%d" % amount, Vector2(900, 260), Color(1.0, 0.75, 0.2))
        "block":
            player.gain_block(amount)
            floating_text_requested.emit("+%d Block" % amount, Vector2(320, 430), Color(0.45, 0.75, 1.0))
        "draw":
            player.draw_cards(amount)
            floating_text_requested.emit("+%d Card" % amount, Vector2(520, 540), Color(0.8, 0.9, 1.0))
        _:
            push_warning("Unknown card effect type: %s" % effect_type)

func _check_battle_end() -> void:
    if enemy.health <= 0:
        is_battle_over = true
        is_player_turn = false
        battle_finished.emit("victory")
    elif player.health <= 0:
        is_battle_over = true
        is_player_turn = false
        battle_finished.emit("defeat")

func _emit_state() -> void:
    state_changed.emit(get_state())
```

- [ ] **Step 6: Run a syntax check**

Run: `godot --headless --path . --check-only --quit`

Expected: Godot exits with code 0 and no parser errors for the five new scripts. If `godot` is not on `PATH`, run the same check from the Godot editor executable available on the machine.

- [ ] **Step 7: Commit**

```bash
git add scripts/cards/CardData.gd scripts/cards/CardLibrary.gd scripts/battle/PlayerState.gd scripts/battle/EnemyState.gd scripts/battle/BattleController.gd
git commit -m "feat: add battle state core"
```

---

### Task 2: Reusable UI Scenes

**Files:**
- Create: `scripts/ui/CardView.gd`
- Create: `scripts/ui/EnemyView.gd`
- Create: `scripts/ui/FloatingText.gd`
- Create: `scenes/CardView.tscn`
- Create: `scenes/EnemyView.tscn`
- Create: `scenes/FloatingText.tscn`

**Interfaces:**
- Consumes: `CardData`
- Produces: `CardView.card_clicked(card: CardData)`
- Produces: `CardView.set_card(card_data: CardData, can_play: bool) -> void`
- Produces: `EnemyView.set_enemy_state(health: int, max_health: int, block: int, intent: Dictionary) -> void`
- Produces: `FloatingText.play(text: String, color: Color) -> void`

- [ ] **Step 1: Write `CardView.gd`**

```gdscript
extends Button
class_name CardView

signal card_clicked(card: CardData)

var card: CardData
var playable := true

@onready var cost_label: Label = %CostLabel
@onready var name_label: Label = %NameLabel
@onready var type_label: Label = %TypeLabel
@onready var description_label: Label = %DescriptionLabel

func _ready() -> void:
    pressed.connect(_on_pressed)
    mouse_entered.connect(_on_mouse_entered)
    mouse_exited.connect(_on_mouse_exited)

func set_card(card_data: CardData, can_play: bool) -> void:
    card = card_data
    playable = can_play
    disabled = not can_play
    cost_label.text = str(card.cost)
    name_label.text = card.display_name
    type_label.text = card.card_type
    description_label.text = card.description
    modulate = Color.WHITE if can_play else Color(0.55, 0.55, 0.55, 1.0)

func _on_pressed() -> void:
    if playable and card != null:
        card_clicked.emit(card)

func _on_mouse_entered() -> void:
    if playable:
        create_tween().tween_property(self, "position:y", position.y - 18.0, 0.08)

func _on_mouse_exited() -> void:
    create_tween().tween_property(self, "position:y", 0.0, 0.08)
```

- [ ] **Step 2: Write `EnemyView.gd`**

```gdscript
extends Control
class_name EnemyView

@onready var health_label: Label = %HealthLabel
@onready var health_bar: ProgressBar = %HealthBar
@onready var block_label: Label = %BlockLabel
@onready var intent_label: Label = %IntentLabel
@onready var enemy_body: ColorRect = %EnemyBody

func set_enemy_state(health: int, max_health: int, block: int, intent: Dictionary) -> void:
    health_bar.max_value = max_health
    health_bar.value = health
    health_label.text = "%d/%d" % [health, max_health]
    block_label.text = "Block %d" % block if block > 0 else ""
    var intent_type := String(intent.get("type", "attack"))
    var amount := int(intent.get("amount", 0))
    match intent_type:
        "attack":
            intent_label.text = "Intent: Attack %d" % amount
        "block":
            intent_label.text = "Intent: Guard %d" % amount
        _:
            intent_label.text = "Intent: %s" % String(intent.get("label", "Unknown"))

func flash_hit() -> void:
    enemy_body.color = Color(1.0, 0.45, 0.35, 1.0)
    var tween := create_tween()
    tween.tween_property(enemy_body, "color", Color(0.18, 0.14, 0.13, 1.0), 0.18)
```

- [ ] **Step 3: Write `FloatingText.gd`**

```gdscript
extends Label
class_name FloatingText

func play(display_text: String, display_color: Color) -> void:
    text = display_text
    modulate = display_color
    var tween := create_tween()
    tween.set_parallel(true)
    tween.tween_property(self, "position:y", position.y - 48.0, 0.55)
    tween.tween_property(self, "modulate:a", 0.0, 0.55)
    tween.set_parallel(false)
    tween.tween_callback(queue_free)
```

- [ ] **Step 4: Create `CardView.tscn`**

```text
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/ui/CardView.gd" id="1"]

[node name="CardView" type="Button"]
custom_minimum_size = Vector2(150, 210)
offset_right = 150.0
offset_bottom = 210.0
focus_mode = 0
script = ExtResource("1")

[node name="Panel" type="Panel" parent="."]
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
mouse_filter = 2

[node name="CostLabel" type="Label" parent="Panel"]
unique_name_in_owner = true
offset_left = 10.0
offset_top = 8.0
offset_right = 42.0
offset_bottom = 38.0
text = "1"

[node name="NameLabel" type="Label" parent="Panel"]
unique_name_in_owner = true
offset_left = 44.0
offset_top = 10.0
offset_right = 142.0
offset_bottom = 34.0
text = "Strike"
horizontal_alignment = 1

[node name="TypeLabel" type="Label" parent="Panel"]
unique_name_in_owner = true
offset_left = 18.0
offset_top = 116.0
offset_right = 132.0
offset_bottom = 138.0
text = "Attack"
horizontal_alignment = 1

[node name="DescriptionLabel" type="Label" parent="Panel"]
unique_name_in_owner = true
offset_left = 14.0
offset_top = 146.0
offset_right = 136.0
offset_bottom = 198.0
text = "Deal 6 damage."
autowrap_mode = 3
horizontal_alignment = 1
```

- [ ] **Step 5: Create `EnemyView.tscn`**

```text
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/ui/EnemyView.gd" id="1"]

[node name="EnemyView" type="Control"]
offset_right = 300.0
offset_bottom = 310.0
script = ExtResource("1")

[node name="IntentLabel" type="Label" parent="."]
unique_name_in_owner = true
offset_left = 48.0
offset_top = 0.0
offset_right = 252.0
offset_bottom = 30.0
text = "Intent: Attack 6"
horizontal_alignment = 1

[node name="EnemyBody" type="ColorRect" parent="."]
unique_name_in_owner = true
offset_left = 70.0
offset_top = 44.0
offset_right = 230.0
offset_bottom = 230.0
color = Color(0.18, 0.14, 0.13, 1)

[node name="HealthBar" type="ProgressBar" parent="."]
unique_name_in_owner = true
offset_left = 30.0
offset_top = 244.0
offset_right = 270.0
offset_bottom = 270.0
max_value = 50.0
value = 50.0

[node name="HealthLabel" type="Label" parent="HealthBar"]
unique_name_in_owner = true
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
text = "50/50"
horizontal_alignment = 1
vertical_alignment = 1

[node name="BlockLabel" type="Label" parent="."]
unique_name_in_owner = true
offset_left = 76.0
offset_top = 276.0
offset_right = 224.0
offset_bottom = 304.0
text = ""
horizontal_alignment = 1
```

- [ ] **Step 6: Create `FloatingText.tscn`**

```text
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/ui/FloatingText.gd" id="1"]

[node name="FloatingText" type="Label"]
offset_right = 160.0
offset_bottom = 36.0
text = "-6"
horizontal_alignment = 1
vertical_alignment = 1
script = ExtResource("1")
```

- [ ] **Step 7: Run a syntax check**

Run: `godot --headless --path . --check-only --quit`

Expected: Godot exits with code 0 and no scene loading or parser errors.

- [ ] **Step 8: Commit**

```bash
git add scripts/ui/CardView.gd scripts/ui/EnemyView.gd scripts/ui/FloatingText.gd scenes/CardView.tscn scenes/EnemyView.tscn scenes/FloatingText.tscn
git commit -m "feat: add battle UI components"
```

---

### Task 3: Main Battle Scene Wiring

**Files:**
- Create: `scenes/BattleScene.tscn`
- Create: `scripts/battle/BattleScene.gd`
- Modify: `project.godot`

**Interfaces:**
- Consumes: `BattleController.state_changed(state: Dictionary)`
- Consumes: `BattleController.floating_text_requested(text: String, world_position: Vector2, color: Color)`
- Consumes: `BattleController.turn_banner_requested(text: String)`
- Consumes: `BattleController.battle_finished(result: String)`
- Consumes: `CardView.card_clicked(card: CardData)`

- [ ] **Step 1: Create `scripts/battle/BattleScene.gd`**

```gdscript
extends Control

const CARD_VIEW_SCENE := preload("res://scenes/CardView.tscn")
const FLOATING_TEXT_SCENE := preload("res://scenes/FloatingText.tscn")

@onready var controller: BattleController = %BattleController
@onready var enemy_view: EnemyView = %EnemyView
@onready var hand_container: HBoxContainer = %HandContainer
@onready var player_health_label: Label = %PlayerHealthLabel
@onready var player_block_label: Label = %PlayerBlockLabel
@onready var energy_label: Label = %EnergyLabel
@onready var pile_label: Label = %PileLabel
@onready var end_turn_button: Button = %EndTurnButton
@onready var turn_banner: Label = %TurnBanner
@onready var overlay: Panel = %Overlay
@onready var overlay_label: Label = %OverlayLabel
@onready var feedback_layer: Control = %FeedbackLayer

func _ready() -> void:
    controller.state_changed.connect(_render_state)
    controller.floating_text_requested.connect(_spawn_floating_text)
    controller.turn_banner_requested.connect(_show_turn_banner)
    controller.battle_finished.connect(_show_battle_result)
    end_turn_button.pressed.connect(controller.end_player_turn)
    _render_state(controller.get_state())

func _render_state(state: Dictionary) -> void:
    player_health_label.text = "HP %d/%d" % [state["player_health"], state["player_max_health"]]
    player_block_label.text = "Block %d" % state["player_block"]
    energy_label.text = "Energy %d" % state["player_energy"]
    pile_label.text = "Draw %d | Discard %d" % [state["draw_count"], state["discard_count"]]
    enemy_view.set_enemy_state(
        state["enemy_health"],
        state["enemy_max_health"],
        state["enemy_block"],
        state["enemy_intent"]
    )
    end_turn_button.disabled = not state["is_player_turn"] or state["is_battle_over"]
    _render_hand(state["hand"], state["player_energy"], state["is_player_turn"], state["is_battle_over"])

func _render_hand(hand: Array, energy: int, is_player_turn: bool, is_battle_over: bool) -> void:
    for child in hand_container.get_children():
        child.queue_free()
    for card: CardData in hand:
        var card_view: CardView = CARD_VIEW_SCENE.instantiate()
        hand_container.add_child(card_view)
        card_view.set_card(card, is_player_turn and not is_battle_over and energy >= card.cost)
        card_view.card_clicked.connect(_on_card_clicked)

func _on_card_clicked(card: CardData) -> void:
    var played := controller.play_card(card)
    if played:
        enemy_view.flash_hit()

func _spawn_floating_text(display_text: String, world_position: Vector2, color: Color) -> void:
    var floating_text: FloatingText = FLOATING_TEXT_SCENE.instantiate()
    feedback_layer.add_child(floating_text)
    floating_text.position = world_position
    floating_text.play(display_text, color)

func _show_turn_banner(display_text: String) -> void:
    turn_banner.text = display_text
    turn_banner.modulate.a = 1.0
    var tween := create_tween()
    tween.tween_interval(0.45)
    tween.tween_property(turn_banner, "modulate:a", 0.0, 0.35)

func _show_battle_result(result: String) -> void:
    overlay.visible = true
    overlay_label.text = "Victory" if result == "victory" else "Defeat"
```

- [ ] **Step 2: Create `BattleScene.tscn` with controller and layout**

```text
[gd_scene load_steps=6 format=3]

[ext_resource type="Script" path="res://scripts/battle/BattleController.gd" id="1"]
[ext_resource type="PackedScene" path="res://scenes/CardView.tscn" id="2"]
[ext_resource type="PackedScene" path="res://scenes/EnemyView.tscn" id="3"]
[ext_resource type="PackedScene" path="res://scenes/FloatingText.tscn" id="4"]
[ext_resource type="Script" path="res://scripts/battle/BattleScene.gd" id="5"]

[node name="BattleScene" type="Control"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
script = ExtResource("5")

[node name="Background" type="ColorRect" parent="."]
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
color = Color(0.055, 0.049, 0.047, 1)

[node name="BattleController" type="Node" parent="."]
unique_name_in_owner = true
script = ExtResource("1")

[node name="PlayerPanel" type="Panel" parent="."]
offset_left = 80.0
offset_top = 300.0
offset_right = 420.0
offset_bottom = 520.0

[node name="PlayerTitle" type="Label" parent="PlayerPanel"]
offset_left = 20.0
offset_top = 16.0
offset_right = 220.0
offset_bottom = 44.0
text = "The Adventurer"

[node name="PlayerHealthLabel" type="Label" parent="PlayerPanel"]
unique_name_in_owner = true
offset_left = 20.0
offset_top = 62.0
offset_right = 220.0
offset_bottom = 90.0
text = "HP 70/70"

[node name="PlayerBlockLabel" type="Label" parent="PlayerPanel"]
unique_name_in_owner = true
offset_left = 20.0
offset_top = 98.0
offset_right = 220.0
offset_bottom = 126.0
text = "Block 0"

[node name="EnergyLabel" type="Label" parent="."]
unique_name_in_owner = true
offset_left = 80.0
offset_top = 548.0
offset_right = 240.0
offset_bottom = 588.0
text = "Energy 3"

[node name="PileLabel" type="Label" parent="."]
unique_name_in_owner = true
offset_left = 250.0
offset_top = 548.0
offset_right = 520.0
offset_bottom = 588.0
text = "Draw 5 | Discard 0"

[node name="EnemyAnchor" type="Control" parent="."]
offset_left = 760.0
offset_top = 120.0
offset_right = 1060.0
offset_bottom = 430.0

[node name="EnemyView" parent="EnemyAnchor" instance=ExtResource("3")]
unique_name_in_owner = true

[node name="HandContainer" type="HBoxContainer" parent="."]
unique_name_in_owner = true
offset_left = 260.0
offset_top = 620.0
offset_right = 1100.0
offset_bottom = 850.0
theme_override_constants/separation = 14

[node name="EndTurnButton" type="Button" parent="."]
unique_name_in_owner = true
offset_left = 1120.0
offset_top = 540.0
offset_right = 1270.0
offset_bottom = 596.0
text = "End Turn"

[node name="TurnBanner" type="Label" parent="."]
unique_name_in_owner = true
offset_left = 500.0
offset_top = 48.0
offset_right = 780.0
offset_bottom = 88.0
text = ""
horizontal_alignment = 1

[node name="Overlay" type="Panel" parent="."]
unique_name_in_owner = true
visible = false
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2

[node name="OverlayLabel" type="Label" parent="Overlay"]
unique_name_in_owner = true
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -180.0
offset_top = -32.0
offset_right = 180.0
offset_bottom = 32.0
grow_horizontal = 2
grow_vertical = 2
text = "Victory"
horizontal_alignment = 1
vertical_alignment = 1

[node name="FeedbackLayer" type="Control" parent="."]
unique_name_in_owner = true
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
mouse_filter = 2
```

- [ ] **Step 3: Set the main scene in `project.godot`**

Add this line under `[application]`:

```text
run/main_scene="res://scenes/BattleScene.tscn"
```

- [ ] **Step 4: Run a syntax and scene load check**

Run: `godot --headless --path . --check-only --quit`

Expected: Godot exits with code 0. There should be no errors about missing scripts, missing scene paths, unknown classes, or invalid unique node references.

- [ ] **Step 5: Run the scene headless briefly**

Run: `godot --headless --path . --quit-after 2`

Expected: Godot starts the project, loads `BattleScene.tscn`, and exits after 2 seconds without script errors.

- [ ] **Step 6: Commit**

```bash
git add scenes/BattleScene.tscn scripts/battle/BattleScene.gd project.godot
git commit -m "feat: wire main battle scene"
```

---

### Task 4: Visual Direction And Generated Asset Integration

**Files:**
- Create: `assets/generated/.gitkeep`
- Create: `docs/art/battle-demo-asset-prompts.md`
- Modify: `scenes/BattleScene.tscn`
- Modify: `scenes/CardView.tscn`
- Modify: `scenes/EnemyView.tscn`

**Interfaces:**
- Consumes: `assets/generated/battle_background.png`
- Consumes: `assets/generated/enemy_cultist.png`
- Consumes: `assets/generated/player_adventurer.png`
- Consumes: `assets/generated/card_frame.png`
- Produces: A scene that still runs when generated assets are unavailable by keeping dark color fallback nodes.

- [ ] **Step 1: Create generated asset directory marker**

Create `assets/generated/.gitkeep` as an empty file.

- [ ] **Step 2: Create `docs/art/battle-demo-asset-prompts.md`**

```markdown
# Battle Demo Asset Prompts

All image generation for this project uses model `gpt-image-2`.

Credentials must stay in local environment variables or ignored local config. Do not commit API keys.

## Battle Background

Dark fantasy hand-painted game background, ruined stone dungeon hall, cold shadows, faint torchlight, readable center stage for a card battle UI, no characters, no text, 16:9 composition, high contrast focal area, painterly texture.

Target file: `assets/generated/battle_background.png`

## Enemy Cultist

Dark fantasy hand-painted enemy character, hooded grave cultist with bone ornaments, readable silhouette, full body, front three-quarter view, transparent background if possible, no text, game sprite concept art.

Target file: `assets/generated/enemy_cultist.png`

## Player Adventurer

Dark fantasy hand-painted armored adventurer silhouette, cloak, sword at side, readable from distance, front three-quarter view, transparent background if possible, no text, game sprite concept art.

Target file: `assets/generated/player_adventurer.png`

## Card Frame

Dark fantasy card frame, worn parchment center, aged iron border, subtle red wax and scratches, empty center for text, no letters, no symbols, vertical playing card frame, readable UI asset.

Target file: `assets/generated/card_frame.png`
```

- [ ] **Step 3: Generate or place assets locally**

Generate the four target files using the configured `gpt-image-2` service. Use local environment variables for credentials. Do not paste credentials into the prompt file, scripts, shell history, or committed files.

Expected files after generation:

```text
assets/generated/battle_background.png
assets/generated/enemy_cultist.png
assets/generated/player_adventurer.png
assets/generated/card_frame.png
```

- [ ] **Step 4: Add background and character TextureRects**

In `scenes/BattleScene.tscn`, add ext resources:

```text
[ext_resource type="Texture2D" path="res://assets/generated/battle_background.png" id="6"]
[ext_resource type="Texture2D" path="res://assets/generated/player_adventurer.png" id="7"]
```

Add a `TextureRect` above the fallback `Background` ColorRect or replace the ColorRect if the asset exists:

```text
[node name="BackgroundArt" type="TextureRect" parent="."]
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
texture = ExtResource("6")
expand_mode = 1
stretch_mode = 6
```

Add player art inside `PlayerPanel`:

```text
[node name="PlayerArt" type="TextureRect" parent="PlayerPanel"]
offset_left = 190.0
offset_top = 18.0
offset_right = 326.0
offset_bottom = 198.0
texture = ExtResource("7")
expand_mode = 1
stretch_mode = 5
```

- [ ] **Step 5: Add enemy art**

In `scenes/EnemyView.tscn`, add:

```text
[ext_resource type="Texture2D" path="res://assets/generated/enemy_cultist.png" id="2"]
```

Add this node above `EnemyBody` or replace `EnemyBody` while keeping `EnemyBody` as hit-flash fallback:

```text
[node name="EnemyArt" type="TextureRect" parent="."]
offset_left = 44.0
offset_top = 30.0
offset_right = 256.0
offset_bottom = 238.0
texture = ExtResource("2")
expand_mode = 1
stretch_mode = 5
```

- [ ] **Step 6: Add card frame art**

In `scenes/CardView.tscn`, add:

```text
[ext_resource type="Texture2D" path="res://assets/generated/card_frame.png" id="2"]
```

Add this as the first child of `CardView`:

```text
[node name="FrameArt" type="TextureRect" parent="."]
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
texture = ExtResource("2")
expand_mode = 1
stretch_mode = 5
mouse_filter = 2
```

Keep labels above the frame so text remains readable.

- [ ] **Step 7: Run scene load check**

Run: `godot --headless --path . --check-only --quit`

Expected: Godot exits with code 0 and imports generated textures without missing-resource errors.

- [ ] **Step 8: Commit**

```bash
git add assets/generated/.gitkeep docs/art/battle-demo-asset-prompts.md scenes/BattleScene.tscn scenes/CardView.tscn scenes/EnemyView.tscn assets/generated/battle_background.png assets/generated/enemy_cultist.png assets/generated/player_adventurer.png assets/generated/card_frame.png
git commit -m "feat: add dark fantasy battle visuals"
```

---

### Task 5: Combat Feedback And Polish

**Files:**
- Modify: `scripts/battle/BattleScene.gd`
- Modify: `scripts/ui/CardView.gd`
- Modify: `scripts/ui/EnemyView.gd`
- Modify: `scenes/BattleScene.tscn`

**Interfaces:**
- Consumes: `BattleController.floating_text_requested`
- Consumes: `EnemyView.flash_hit() -> void`
- Produces: stable visible feedback for card play, enemy hit, enemy turn, victory, and defeat.

- [ ] **Step 1: Improve card hover so cards return to their original slot**

Replace `CardView.gd` with:

```gdscript
extends Button
class_name CardView

signal card_clicked(card: CardData)

var card: CardData
var playable := true
var base_position := Vector2.ZERO

@onready var cost_label: Label = %CostLabel
@onready var name_label: Label = %NameLabel
@onready var type_label: Label = %TypeLabel
@onready var description_label: Label = %DescriptionLabel

func _ready() -> void:
    base_position = position
    pressed.connect(_on_pressed)
    mouse_entered.connect(_on_mouse_entered)
    mouse_exited.connect(_on_mouse_exited)

func set_card(card_data: CardData, can_play: bool) -> void:
    card = card_data
    playable = can_play
    disabled = not can_play
    cost_label.text = str(card.cost)
    name_label.text = card.display_name
    type_label.text = card.card_type
    description_label.text = card.description
    modulate = Color.WHITE if can_play else Color(0.55, 0.55, 0.55, 1.0)

func _on_pressed() -> void:
    if playable and card != null:
        card_clicked.emit(card)

func _on_mouse_entered() -> void:
    if playable:
        base_position = position
        var tween := create_tween()
        tween.set_parallel(true)
        tween.tween_property(self, "position", base_position + Vector2(0, -18), 0.08)
        tween.tween_property(self, "scale", Vector2(1.06, 1.06), 0.08)

func _on_mouse_exited() -> void:
    var tween := create_tween()
    tween.set_parallel(true)
    tween.tween_property(self, "position", base_position, 0.08)
    tween.tween_property(self, "scale", Vector2.ONE, 0.08)
```

- [ ] **Step 2: Add scene shake on heavy hits**

In `BattleScene.gd`, add:

```gdscript
func _shake_scene(strength: float = 8.0) -> void:
    var original_position := position
    var tween := create_tween()
    tween.tween_property(self, "position", original_position + Vector2(strength, 0), 0.04)
    tween.tween_property(self, "position", original_position + Vector2(-strength, 0), 0.04)
    tween.tween_property(self, "position", original_position, 0.04)
```

Modify `_on_card_clicked`:

```gdscript
func _on_card_clicked(card: CardData) -> void:
    var played := controller.play_card(card)
    if played:
        enemy_view.flash_hit()
        if card.id == "heavy_blow":
            _shake_scene(10.0)
```

- [ ] **Step 3: Make victory and defeat overlays readable**

In `scenes/BattleScene.tscn`, update `Overlay` and `OverlayLabel`:

```text
[node name="Overlay" type="Panel" parent="."]
unique_name_in_owner = true
visible = false
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
modulate = Color(0, 0, 0, 0.86)

[node name="OverlayLabel" type="Label" parent="Overlay"]
unique_name_in_owner = true
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -220.0
offset_top = -50.0
offset_right = 220.0
offset_bottom = 50.0
grow_horizontal = 2
grow_vertical = 2
text = "Victory"
horizontal_alignment = 1
vertical_alignment = 1
```

- [ ] **Step 4: Add enemy intent pulse before enemy action**

In `EnemyView.gd`, add:

```gdscript
func pulse_intent() -> void:
    var tween := create_tween()
    tween.tween_property(intent_label, "scale", Vector2(1.16, 1.16), 0.12)
    tween.tween_property(intent_label, "scale", Vector2.ONE, 0.12)
```

In `BattleScene.gd`, modify `_show_turn_banner`:

```gdscript
func _show_turn_banner(display_text: String) -> void:
    turn_banner.text = display_text
    turn_banner.modulate.a = 1.0
    if display_text == "Enemy Turn":
        enemy_view.pulse_intent()
    var tween := create_tween()
    tween.tween_interval(0.45)
    tween.tween_property(turn_banner, "modulate:a", 0.0, 0.35)
```

- [ ] **Step 5: Run scene load check**

Run: `godot --headless --path . --check-only --quit`

Expected: Godot exits with code 0 and no parser errors.

- [ ] **Step 6: Commit**

```bash
git add scripts/battle/BattleScene.gd scripts/ui/CardView.gd scripts/ui/EnemyView.gd scenes/BattleScene.tscn
git commit -m "feat: polish battle feedback"
```

---

### Task 6: Manual Verification Checklist And Final Pass

**Files:**
- Create: `tests/manual/battle-demo-checklist.md`
- Modify if needed: any file touched by prior tasks to fix verification failures.

**Interfaces:**
- Consumes: completed playable scene.
- Produces: checked manual verification document.

- [ ] **Step 1: Create `tests/manual/battle-demo-checklist.md`**

```markdown
# Battle Demo Manual Verification

Run command:

```bash
godot --path .
```

## Required Checks

- [ ] Project starts directly in `scenes/BattleScene.tscn`.
- [ ] Player starts with 70 health, 0 block, 3 energy, and 5 cards in hand.
- [ ] Enemy starts with 50 health and visible intent.
- [ ] Strike costs 1 energy and deals 6 damage.
- [ ] Defend costs 1 energy and gives 5 block.
- [ ] Heavy Blow costs 2 energy and deals 12 damage.
- [ ] Quick Thought costs 0 energy and draws 1 card.
- [ ] Cards with insufficient energy are disabled.
- [ ] End Turn discards the remaining hand.
- [ ] Enemy attack removes block before health.
- [ ] Draw pile reshuffles from discard when empty.
- [ ] Victory overlay appears when enemy health reaches 0.
- [ ] Defeat overlay appears when player health reaches 0.
- [ ] Floating text appears for damage, block, and draw.
- [ ] Enemy intent is readable before enemy action.
- [ ] Battle screen uses dark fantasy generated assets or dark fantasy fallback visuals.
- [ ] No API key or secret appears in tracked files.
```
```

- [ ] **Step 2: Run tracked-secret scan**

Run: `rg -n "sk-|api[_-]?key|OPENAI|base_url|image generation credential" . --glob '!docs/superpowers/plans/2026-06-26-battle-demo-implementation.md'`

Expected: No API key values are printed. A mention of an image-generation base URL in local ignored config is acceptable only if the file is not tracked.

- [ ] **Step 3: Run syntax check**

Run: `godot --headless --path . --check-only --quit`

Expected: Godot exits with code 0.

- [ ] **Step 4: Run the project**

Run: `godot --path .`

Expected: Godot opens the battle scene. Complete every checklist item in `tests/manual/battle-demo-checklist.md`.

- [ ] **Step 5: Fix verification failures**

For any failed checklist item, make the smallest code or scene change that corrects the behavior, then rerun:

```bash
godot --headless --path . --check-only --quit
godot --path .
```

Expected: all required checklist items pass.

- [ ] **Step 6: Commit**

```bash
git add tests/manual/battle-demo-checklist.md scripts scenes project.godot assets/generated docs/art
git commit -m "test: add battle demo verification checklist"
```

---

## Plan Self-Review

- Spec coverage: Tasks cover state core, one-player one-enemy battle flow, four required card types, enemy intent, UI, generated dark fantasy assets, feedback, main scene startup, invalid play rejection, and manual verification.
- Scope check: The plan does not add map flow, rewards, relics, potions, saves, settings, or multi-enemy combat.
- Placeholder scan: No `TBD`, `TODO`, or unspecified implementation steps remain. Optional items from the design are either excluded or explicitly non-blocking.
- Type consistency: `CardData`, `CardLibrary`, `PlayerState`, `EnemyState`, `BattleController`, `CardView`, `EnemyView`, and `FloatingText` names and method signatures are consistent across tasks.
- Risk note: Godot `.tscn` files are sensitive to exact formatting and generated UIDs. If manual scene text causes loader issues, open and save the scenes in Godot 4.7 or run the Godot UID update tool before continuing.
