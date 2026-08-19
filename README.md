# ♜ Risk Zone Chess

An experimental chess variant combining standard chess movement with a dynamic environmental hazard system and a Stockfish-powered **Trajectory Risk Decision AI**.
The board is familiar, the pieces move according to standard chess rules, and checkmate remains the ultimate goal — but two central rows form the **Risk Zone**, where pieces can be suddenly engulfed and removed from play.

 **The center of the battlefield is dangerous.**. This creates a game where **strategy, tactics, positioning, and risk management** are equally important.

The project is implemented as a self-contained HTML5 application with Vanilla JavaScript. It uses `chess.js` for chess state and legal move handling, while Stockfish runs in a Web Worker when available.

---

## 1. Key Features

- **Dynamic Risk Zone:** Ranks **4 and 5** form a 16-square Risk Zone.
- **Random Hazard Strike:** After every valid move, one Risk Zone square is selected at random.
- **Piece Destruction:** If the selected square contains a piece, that piece is removed from the board.
- **King Hazard:** If a King is swallowed by the Risk Zone, its owner immediately loses.
- **Pinned-Piece Hazard:** If a pinned piece is swallowed, the game immediately ends in a draw.
- **Stockfish AI:** Uses Stockfish through a Web Worker and UCI protocol, with WebAssembly support through the local Stockfish build.
- **MultiPV Analysis:** Stockfish can provide up to **3 candidate principal variations**.
- **Trajectory Risk Evaluation:** The Bot evaluates future Bot moves for exposure to the Risk Zone over a horizon of **6 plies**.
- **Discounted Risk:** Future trajectory risk is discounted with **γ = 0.6**.
- **Responsive Interface:** Includes legal-move indicators, check indicators, last-move highlighting, temporary hazard reveals, captured-piece displays, difficulty controls, and a live MultiPV prediction log.
- **Direct Launch Support:** The application includes a local/CDN Worker fallback intended to make the HTML usable when opened directly through `file://`.

---

## 2. Rules of the Game

The game keeps standard chess movement and legal-move rules through `chess.js`, but adds the Risk Zone hazard mechanics.

### 2.1 Risk Zone

Ranks **4 and 5** are dangerous:

```text
8 | r | n | b | q | k | b | n | r |  Black
7 | p | p | p | p | p | p | p | p |  |
6 | . | . | . | . | . | . | . | . |  |
5 | * | * | * | * | * | * | * | * |  <- RISK ZONE
4 | * | * | * | * | * | * | * | * |  <- RISK ZONE
3 | . | . | . | . | . | . | . | . |  |
2 | P | P | P | P | P | P | P | P |  |
1 | R | N | B | Q | K | B | N | R |  White
    a   b   c   d   e   f   g   h
```

The Risk Zone therefore contains:

- `a4`–`h4`
- `a5`–`h5`
- **16 squares in total**

### 2.2 Hazard Strike

Immediately after every valid White or Black move:

1. One of the 16 Risk Zone squares is selected randomly.
2. If the selected square is empty, nothing is destroyed.
3. If a piece occupies the selected square, it is swallowed and removed from play.
4. If the swallowed piece is a **King**, the owner of that King loses immediately.
5. If the swallowed piece is **pinned**, the game immediately ends in a **draw**.

The selected square is normally hidden. When a piece is swallowed, the affected square is temporarily revealed for that event; the reveal is cleared when the next move begins.

### 2.3 Check 

The implementation retains normal check/checkmate/stalemate handling through `chess.js`.

### 2.4 Draw Conditions

The implementation also ends the game as a draw for:

- **Stalemate**
- **Insufficient material**
- **Fifty-move rule**, represented by a halfmove clock of at least 100
- **Pinned piece swallowed by the Risk Zone**

---

## 3. AI Engine Architecture

### 3.1 Stockfish Loading and Fallback

The application uses a three-stage practical fallback strategy:

```text
                 +----------------------+
                 |  Initialize Engine   |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | Local Stockfish      |
                 | Web Worker           |
                 +----------+-----------+
                            |
                    failure / file://
                            |
                            v
                 +----------------------+
                 | CDN Blob Worker      |
                 +----------+-----------+
                            |
                         failure
                            |
                            v
                 +----------------------+
                 | MOCK Candidate       |
                 | Generator            |
                 +----------------------+
```

#### Local Stockfish Worker

The code first supports the local worker:

```text
stockfish/stockfish-18-lite-single.js
```

The local worker is used when the page is served through a normal web context.

#### CDN Blob Worker

When appropriate, including direct `file://` usage or local-worker failure, the application creates a Worker from a Blob containing an `importScripts()` call to the configured Stockfish CDN resource.

The current code uses:

```text
https://cdn.jsdelivr.net/npm/stockfish.js@10.0.2/stockfish.js
```

This fallback therefore requires network access.

#### MOCK Mode

If Stockfish cannot be used, the application falls back to a lightweight heuristic candidate generator. It evaluates legal moves using captured-piece value plus a small random component and keeps up to three candidates.

MOCK mode is a responsiveness fallback; it is **not equivalent to Stockfish strength**.

---

## 4. Trajectory Risk Decision Engine

Stockfish provides conventional chess evaluations in centipawns. The custom decision layer then considers the additional environmental risk introduced by the Risk Zone.

### 4.1 MultiPV Candidates

The application requests up to **3 MultiPV candidates** from Stockfish.

For each candidate, its principal variation is examined.

### 4.2 Prediction Horizon

The trajectory evaluator examines up to:

```text
6 plies
```

Only **Bot moves** within that trajectory contribute to the trajectory risk score.

### 4.3 Step Risk

A Bot move receives a risk penalty when its destination lies in the Risk Zone.

The actual penalties implemented in the code are:

| Piece | Risk Penalty |
|---|---:|
| King | 5000 |
| Queen | 280 |
| Rook | 150 |
| Bishop | 100 |
| Knight | 95 |
| Pawn | 30 |

These are trajectory penalties, not the ordinary chess piece values used by the MOCK generator.

### 4.4 Discounted Cumulative Risk

The risk of future Bot moves is discounted exponentially:

```text
CumulativeRisk = Σ (γ^i × StepRisk_i)
```

with:

```text
γ = 0.6
```

The implementation starts with a weight of `1.0` and multiplies the weight by `0.6` after each simulated ply.

### 4.5 Candidate Selection

For each candidate:

```text
CPGap = max(0, maxCP - candidateCP)

FinalScore = -CPGap - (0.4 × CumulativeRisk)
```

The candidate with the highest `FinalScore` is selected.

This means the Bot does not simply choose the Stockfish candidate with the highest raw evaluation. It attempts to balance chess strength against predicted exposure to the Risk Zone.

---

## 5. Difficulty Levels

The application provides three difficulty settings.

| Level | Stockfish Strength | Move Time | MultiPV | Description |
|---|---|---:|---:|---|
| **Easy** | UCI Elo 1320 | 500 ms | 1 | Throttled Stockfish at the lowest native Elo target used by the build |
| **Medium** | UCI Elo 1700 | 1000 ms | 2 | Throttled Stockfish plus trajectory prediction |
| **Hard** | No strength limit | 1800 ms | 3 | Full-strength Stockfish plus trajectory optimization |

### Notes

- Easy and Medium use `UCI_LimitStrength`.
- Easy targets **1320 Elo**.
- Medium targets **1700 Elo**.
- Hard disables `UCI_LimitStrength` and sets Stockfish `Skill Level` to 20.
- The trajectory engine itself uses the common six-ply horizon and `γ = 0.6`; the UI description for Medium refers to the additional prediction layer rather than a different global horizon.

---

## 6. Code Architecture

The application is organized into three logical layers.

```text
+---------------------------------------------------------------+
|                         UI / DOM                              |
|  Board Renderer | Status | Difficulty | Captured Pieces | Log |
+-------------------------------+-------------------------------+
                                |
                                v
+---------------------------------------------------------------+
|                       Game Logic                              |
|     chess.js | King Geometry | Pin Detection | Risk Engine   |
+-------------------------------+-------------------------------+
                                |
                                v
+---------------------------------------------------------------+
|                 AI / Decision Layer                           |
| Stockfish Worker | UCI/MultiPV Parser | Trajectory Evaluator |
|                         |                                     |
|                    MOCK fallback                              |
+---------------------------------------------------------------+
```

### Main components

- **Board renderer:** Draws the chessboard, pieces, legal moves, check state, last move and temporary Risk Zone reveal.
- **Game logic:** Maintains the `chess.js` game state and validates normal chess moves.
- **King geometry:** Provides additional geometric attack detection.
- **Hazard engine:** Randomly selects and resolves Risk Zone strikes.
- **Stockfish Worker:** Communicates with Stockfish using UCI commands.
- **MultiPV parser:** Extracts candidate scores and principal variations from Stockfish `info` messages.
- **Trajectory evaluator:** Scores candidate lines according to future Risk Zone exposure.
- **MOCK generator:** Supplies simple fallback candidates when Stockfish is unavailable.
- **UI logger:** Displays engine status, moves, trajectory analysis and hazard events.

---

## 7. Important Functions

Some of the main functions in the current implementation are:

```text
triggerRiskZone()
```

Selects a random Risk Zone square and resolves the environmental hazard.

```text
calculateStepRisk()
```

Calculates the Risk Zone penalty for a move based on the moving piece.

```text
evaluateCandidatePV()
```

Simulates a Stockfish principal variation and calculates discounted trajectory risk.

```text
chooseRiskAwareMove()
```

Compares legal candidates and selects the candidate with the highest final decision score.

```text
askStockfish()
```

Sends the current FEN and engine settings to Stockfish and retrieves MultiPV candidates.

```text
getMockCandidates()
```

Generates lightweight fallback candidates when Stockfish is unavailable.

```text
isPinned()
```

Determines whether removing a piece exposes a new attack on its King.

```text
checkBoardStatus()
```

Handles checkmate, stalemate, insufficient material, fifty-move draw detection and check messages.

---

## 8. GitHub Pages

This repository is structured as a static website:

```text
index.html
```

is the GitHub Pages entry point.

GitHub Pages can publish static HTML, CSS and JavaScript files directly from a repository. For this project, no server-side runtime is required.

The local Stockfish files are referenced from:

```text
stockfish/stockfish-18-lite-single.js
stockfish/stockfish-18-lite-single.wasm
```

The application still contains its CDN fallback for cases where the local Worker cannot be started.

## 9. Quick Start

### 9.1 Direct Launch

The application is designed to support a zero-installation workflow.

Place these files together:

```text
index.html
stockfish/
├── stockfish/stockfish-18-lite-single.js
└── stockfish-18-lite-single.wasm
```

Then open the HTML file in a modern browser.

The application attempts to use the local Stockfish Worker where supported. If direct `file://` execution prevents the local Worker from loading, the code attempts its CDN Blob Worker fallback.

> **Offline note:** The CDN fallback requires Internet access. If both the local Worker and CDN fallback are unavailable, the application switches to MOCK mode.

### 9.2 Local HTTP Server

For a more conventional browser environment, use a local HTTP server.

#### Python 3

```bash
python -m http.server 8000
```

Then open:

```text
http://localhost:8000/
```

#### Node.js

```bash
npx http-server -p 8000
```

---

## 10. Project Structure

A minimal local installation can contain:

```text
Risk-Zone-Chess/
├── index.html
├── stockfish/stockfish-18-lite-single.js
└── stockfish-18-lite-single.wasm
```

The HTML file contains the application UI, CSS and JavaScript logic. Stockfish is loaded as a Web Worker.

---

## 11. Dependencies

### Runtime

- **HTML5**
- **CSS3**
- **Vanilla JavaScript (ES6+)**
- **chess.js 0.10.3**
- **Stockfish JavaScript/WebAssembly build**

The HTML currently loads `chess.js` 0.10.3 from cdnjs:

```text
https://cdnjs.cloudflare.com/ajax/libs/chess.js/0.10.3/chess.min.js
```

The local Stockfish Worker filename expected by the application is:

```text
stockfish/stockfish-18-lite-single.js
```

The corresponding WebAssembly file should remain available to that Stockfish build.

---

## 12. Browser and Execution Notes

The application uses:

- Web Workers
- Blob URLs
- WebAssembly-backed Stockfish
- CDN resources
- modern JavaScript APIs

A recent desktop browser such as Chrome, Edge or Firefox is recommended.

When running completely offline, the local Stockfish files and the externally loaded `chess.js` dependency must both be available locally for fully offline operation. In the current HTML, `chess.js` is loaded from cdnjs, so a network connection is normally required for that dependency.

---

## 13. AI Decision Log

The right-side **Prediction log (MultiPV)** reports information such as:

- Stockfish candidate moves
- Centipawn evaluations
- Risk trajectory penalties
- Principal variations
- The selected compromise or safe trajectory

Example format:

```text
1. e2→e4 (+0.35) | Risk Traj: -0 ✓ Safe trajectory
   PV: e2e4 ...
```

When a candidate exposes a valuable piece to the Risk Zone, the log can show the discounted trajectory contribution.

---

## 14. Design Philosophy

Risk Zone Chess separates two concepts:

1. **Chess evaluation** — provided primarily by Stockfish.
2. **Environmental survival** — evaluated by the custom trajectory-risk layer.

The Bot therefore tries to find a move that is both strategically strong and less vulnerable to future random destruction.

The goal is not simply to make Stockfish play conventional chess. It is to adapt conventional engine analysis to the game's environmental hazard system.

---

## 15. Current Limitations

- The Risk Zone is fixed to ranks 4 and 5.
- Hazard selection is random and not player-controlled.
- The trajectory model only penalizes future **Bot** moves in the evaluated PV.
- The trajectory horizon is fixed at 6 plies.
- The risk discount factor is fixed at 0.6.
- The Risk Zone penalties are hand-defined constants.
- MOCK mode is deliberately simple and does not reproduce Stockfish analysis quality.
- The CDN fallback depends on network availability.

---

## 16. Future Work

Possible extensions include:

- Configurable Risk Zone layouts.
- Additional hazard types.
- Configurable trajectory horizon and discount factor.
- Calibration of Risk Zone penalties.
- Improved offline packaging of all dependencies.
- More sophisticated hazard-aware search.
- Additional difficulty profiles.
- Statistics and replay support.
- Human-vs-human mode.
- Bot-vs-bot mode.
- More advanced visualization of predicted trajectories.

---

## 17. License


This project is licensed under the MIT License - see the LICENSE file for details.

---

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

## 📫 Contact

For questions, suggestions, or feedback, please open an issue on GitHub.

---

**Enjoy the game, and may the Risk Zone be kind to your pieces! ♟️☠️**
