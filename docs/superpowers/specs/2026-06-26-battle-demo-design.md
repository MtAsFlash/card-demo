# Main Battle Scene Demo Design

## Context

This project is a fresh Godot project for a card battler inspired by Slay the Spire. The current goal is not to build a full roguelike deckbuilder. The goal is to create a playable main battle scene demo that proves the core combat loop and looks convincing enough to present as the foundation of a larger game.

The selected direction is gameplay prototype first, with enough visual polish that the demo reads as a real dark, hand-painted card battle rather than a raw systems test.

## Goals

- Build one complete battle scene with a player, one enemy, a hand of cards, energy, health, block, draw pile, discard pile, and turn progression.
- Make the first version visually credible: dark fantasy mood, readable card UI, clear enemy intent, visible combat feedback, and polished enough layout rhythm.
- Keep combat rules deliberately small so the team can later expand card effects, enemies, relics, rewards, and map flow without rewriting the demo.
- Use generated bitmap assets for the battle background, enemy art, player figure, card frames, and simple icons.

## Non-Goals

- No map, run progression, event rooms, shops, campfires, rewards, relics, potions, or deckbuilding outside the battle.
- No complex status system beyond the minimum needed for the demo.
- No procedural encounter generation.
- No large content set. The demo needs a small curated deck, not a scalable card library yet.
- No online features, saves, localization, or settings menu.

## Target Experience

The player should enter the scene and immediately understand that this is a card-based turn battle. The battle needs to communicate five things clearly:

- The player's current health, block, and energy.
- The enemy's current health and next intent.
- Which cards can be played this turn.
- What happened after a card was played.
- When the turn changes, and whether the battle was won or lost.

The demo should feel close to the structure of Slay the Spire's battle screen, but with its own dark fantasy treatment. The visual target is heavy, moody, hand-painted fantasy: muted background, strong silhouettes, parchment or worn-metal card surfaces, readable icons, and restrained effects.

## Recommended Approach

Build a vertical slice with a small but complete battle loop:

1. Implement the battle loop with simple placeholder visuals.
2. Implement the minimum card data and card effect system.
3. Add one enemy with two or three predictable intents.
4. Build the final battle layout and card interaction.
5. Replace placeholder visuals with generated dark fantasy assets.
6. Add feedback: hover states, play animation, damage numbers, block changes, hit flashes, screen shake, and victory/defeat overlays.

This balances short-term presentation with enough architecture to continue development later.

## Scene Structure

The main battle scene should be `scenes/BattleScene.tscn`.

Primary scene areas:

- Enemy area: enemy art, health bar, block value if needed, intent icon, intent value.
- Player area: player figure or silhouette, health, block.
- Hand area: cards displayed in an arc or horizontal fan near the bottom.
- Resource area: energy indicator, draw pile count, discard pile count, exhaust pile count if implemented later.
- Command area: end turn button and simple victory/defeat overlay.
- Feedback layer: floating text, hit flash, screen shake, and turn banner.

Supporting scenes:

- `scenes/CardView.tscn`: one card view with title, cost, type/icon, description, and playable/disabled states.
- `scenes/EnemyView.tscn`: enemy visual, health display, block display, and intent display.
- `scenes/FloatingText.tscn`: reusable combat number feedback.

## Runtime Modules

Suggested scripts:

- `scripts/battle/BattleController.gd`: owns the combat state machine and turn transitions.
- `scripts/battle/PlayerState.gd`: tracks player health, block, energy, draw pile, hand, discard pile.
- `scripts/battle/EnemyState.gd`: tracks enemy health, block, and the enemy intent queue.
- `scripts/cards/CardData.gd`: defines card name, cost, type, description, targeting mode, and effect list.
- `scripts/cards/CardEffect.gd`: applies simple effects such as damage, block, draw, and temporary energy.
- `scripts/ui/CardView.gd`: renders one card and handles hover/click/drag state.
- `scripts/ui/EnemyView.gd`: renders enemy state and intent.

These boundaries keep the battle rules separate from the UI. UI nodes should ask the controller to play cards or end turns, but they should not directly mutate health, block, or piles.

## Battle Flow

Initial battle setup:

1. Create player state with max health, starting health, deck list, empty hand, empty discard pile, and starting energy rules.
2. Create one enemy with max health and a fixed intent pattern.
3. Shuffle the deck into the draw pile.
4. Start the first player turn.

Player turn:

1. Reset player block.
2. Set energy to 3.
3. Draw 5 cards.
4. Allow playable cards to be selected while energy is available.
5. Apply card effects, then move played cards to discard.
6. End the turn by discarding the remaining hand.

Enemy turn:

1. Resolve the currently displayed enemy intent.
2. Damage first removes player block, then health.
3. Advance to the next enemy intent.
4. If the player is alive, start the next player turn.

Battle end:

- If enemy health reaches 0, show a victory overlay and stop accepting card input.
- If player health reaches 0, show a defeat overlay and stop accepting card input.

## Card Set

The first deck should contain 8 to 12 cards. A simple 10-card deck is enough:

- 4x Strike: 1 energy, deal 6 damage.
- 4x Defend: 1 energy, gain 5 block.
- 1x Heavy Blow: 2 energy, deal 12 damage.
- 1x Quick Thought: 0 energy, draw 1 card.

Optional extra cards if the first four feel too thin:

- Guard Up: 1 energy, gain 8 block.
- Dark Slash: 1 energy, deal 4 damage twice.
- Focus: 1 energy, gain 1 temporary energy this turn.

The demo should start with the smaller set and add optional cards only after the base loop works.

## Enemy Design

Use one enemy for the demo, such as a dark cultist, grave knight, or dungeon beast.

Minimum enemy data:

- Max health: 45 to 60.
- Intent pattern:
  - Attack for 6.
  - Attack for 9.
  - Defend or charge.
- Intent display: icon plus number.

The first implementation can use a fixed repeating pattern. Random enemy AI should wait until the deterministic demo is stable.

## Data Design

Card data can start in code as resource objects or dictionaries. The first demo should favor clarity over a custom editor pipeline.

Each card needs:

- Id
- Display name
- Cost
- Type
- Description
- Targeting mode
- Effects
- Art reference or frame style

Each effect needs:

- Effect type
- Numeric amount
- Optional target

Example effect list:

```text
Strike: [{ type = "damage", amount = 6, target = "enemy" }]
Defend: [{ type = "block", amount = 5, target = "player" }]
Quick Thought: [{ type = "draw", amount = 1, target = "player" }]
```

This is intentionally small. It supports the demo without pretending to be the final card scripting language.

## Visual Asset Plan

Generated image assets should use the configured image generation service with model `gpt-image-2`. API credentials must stay outside git, preferably in environment variables or an ignored local config file.

Needed assets:

- Battle background: dark dungeon, cathedral, ruined hall, or torch-lit stone chamber.
- Enemy: one readable dark fantasy enemy with transparent or easy-to-mask background.
- Player figure: armored adventurer silhouette or half-body portrait.
- Card frame: worn parchment, iron, and shadowed fantasy treatment.
- Intent icons: attack, block, charge.
- Basic effect sprites or particles: slash, hit flash, block shimmer.

Asset rules:

- Prioritize readability over detail.
- Keep cards legible at final in-game size.
- Avoid making the whole screen too dark; important combat UI must remain high contrast.
- Store generated assets under `assets/generated/`.
- Do not commit secrets or prompts containing secrets.

## Interaction Model

The first playable build can support click-to-play. Drag-to-play is desirable but should not block the demo.

Card states:

- Normal: affordable and playable.
- Hovered: lifted or enlarged slightly.
- Disabled: insufficient energy or invalid state.
- Played: quick movement toward the target, then discard.

If click-to-play is used, attacks target the single enemy automatically. This avoids target-selection complexity until multi-enemy combat exists.

## Feedback Requirements

Minimum feedback:

- Card hover lift.
- Energy decreases immediately after a card is played.
- Enemy health bar changes after damage.
- Player block increases after defense cards.
- Floating damage and block numbers.
- Enemy flashes or shakes when hit.
- Turn banner for "Player Turn" and "Enemy Turn".
- Victory and defeat overlays.

Nice-to-have feedback:

- Small screen shake on heavy attacks.
- Different card play sounds or temporary placeholder audio.
- Enemy intent pulse before enemy action.
- Card fan re-layout animation after a card leaves hand.

## Error Handling

The battle controller should reject invalid actions:

- Card not in hand.
- Not the player's turn.
- Not enough energy.
- Battle already ended.
- Missing or unknown card effect type.

In the demo, these can log warnings and leave state unchanged. They should not crash the scene during normal play.

## Testing And Verification

Manual verification is acceptable for the first visual prototype, but the combat rules should be simple enough to unit test later.

Minimum verification pass:

- Start battle and draw 5 cards.
- Play an attack card and confirm energy, discard pile, and enemy health update.
- Play a defend card and confirm block updates.
- End turn and confirm remaining hand is discarded.
- Enemy attacks and damage respects block before health.
- Draw pile reshuffles from discard when needed.
- Victory overlay appears when enemy health reaches 0.
- Defeat overlay appears when player health reaches 0.
- Disabled cards cannot be played when energy is insufficient.

## Milestones

### Milestone 1: Combat Loop Graybox

- Battle scene opens.
- Player and enemy state are visible.
- Cards can be drawn, played, and discarded.
- End turn runs enemy action.
- Victory and defeat states work.

### Milestone 2: Card UI And Interaction

- Card view is readable and reusable.
- Cards show cost, title, description, and disabled state.
- Cards can be clicked to play.
- Hand layout updates after draw, play, and discard.

### Milestone 3: Visual Pass

- Background, enemy, player figure, and card frame are added.
- UI colors and typography match the dark fantasy direction.
- Basic hit/block/turn feedback is added.

### Milestone 4: Demo Polish

- Full battle can be played repeatedly without state bugs.
- Combat feedback is readable.
- Layout holds at the chosen project resolution.
- The demo starts directly in the battle scene.

## Acceptance Criteria

The demo is complete when:

- A player can complete a full one-enemy battle from scene start to victory or defeat.
- The game has a clear player turn and enemy turn loop.
- At least four card types work: attack, block, heavy attack, and draw.
- Energy, health, block, hand, draw pile count, and discard pile count update correctly.
- Enemy intent is visible before the enemy acts.
- The battle screen has generated dark fantasy visual assets and no longer looks like a raw placeholder prototype.
- Invalid card plays are blocked without breaking combat state.

