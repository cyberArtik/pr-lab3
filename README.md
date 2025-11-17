Laboratory Work #3 - Memory Scramble 
Student: Ilico Artemie
Group: FAF-231 

Executive Summary
This laboratory assignment involved building a concurrent, web-based Memory card-matching game where multiple players can interact with the board simultaneously without taking turns. The implementation handles complex asynchronous operations, prevents deadlocks, and provides real-time updates to all connected players through HTTP endpoints.

Table of Contents

Project Architecture
Core Implementation
Containerization Strategy
Quality Assurance
Game Mechanics Verification
Advanced Features


Project Architecture
Directory Structure Analysis
The project follows a modular TypeScript architecture with clear separation of concerns:
lab3_pr/
├── src/              # Core application logic
│   ├── board.ts      # Game state management + concurrency control
│   ├── player.ts     # Player state tracking ADT
│   ├── commands.ts   # High-level game operations
│   ├── server.ts     # HTTP REST API implementation
│   └── simulation.ts # Automated testing scenarios
├── test/             # Unit + integration tests
│   └── board.test.ts # Comprehensive test suite
├── boards/           # Game board configurations
│   ├── perfect.txt   # 3×3 emoji board
│   ├── ab.txt        # 5×5 alternating pattern
│   └── zoom.txt      # 5×5 vehicle emojis
├── public/           # Client-side web interface
│   └── index.html    # Browser-based game UI
├── doc/              # Auto-generated API documentation
└── dist/             # Compiled JavaScript output
Key Design Decision: Separating commands.ts from board.ts allows the server to call high-level operations without managing low-level board state directly.
Show Image

Core Implementation
The Board ADT: Concurrency Model
The Board class uses a sophisticated deferred promise pattern to handle multiple concurrent players:
Data Structures:
typescriptclass Board {
    private readonly cards: (string | null)[][];           // Card pictures
    private readonly faceUp: boolean[][];                  // Visibility state
    private readonly controller: (string | null)[][];      // Ownership tracking
    private readonly waitingForControl: Map<string, Deferred<void>[]>; // Blocked players
    private readonly lingering: Map<string, Array<{row, col}>>;        // Deferred cleanup
    private readonly changeResolvers: Map<string, Function[]>;         // Watch callbacks
}
Critical Insight: The waitingForControl map stores queues of promises for each card position, allowing multiple players to wait concurrently without polling.
The Deferred Class: Custom Promise Management
typescriptclass Deferred<T> {
    public readonly promise: Promise<T>;
    public resolve!: (value: T) => void;
    public reject!: (reason?: unknown) => void;

    public constructor() {
        this.promise = new Promise<T>((resolve, reject) => {
            this.resolve = resolve;
            this.reject = reject;
        });
    }
}
This pattern enables external promise resolution, crucial for waking up waiting players when cards become available.
Player State Management
The Player ADT tracks individual player progress through the two-card flip sequence:
typescriptexport class Player {
    private flips: number;                                    // Statistics
    private firstCard: { row: number; col: number } | null;   // Current play state
    private secondCard: { row: number; col: number } | null;  // Current play state
}
Why two separate card fields? The game rules distinguish between first and second card flips with different behaviors (waiting vs. immediate failure).

Containerization Strategy
Docker Configuration
My Docker setup ensures consistent deployment across different environments:
Dockerfile:
dockerfileFROM node:22.12
WORKDIR /app
COPY package*.json ./

# Install dependencies first (layer caching optimization)
RUN npm install

COPY . .

# Compile TypeScript to JavaScript
RUN npm run compile

# Expose HTTP port
EXPOSE 8080

# Start server with source map support for debugging
CMD ["node", "--require", "source-map-support/register", "dist/src/server.js", "8080", "boards/perfect.txt"]
Key Optimization: Copying package.json before source code allows Docker to cache the npm install layer, speeding up rebuilds.
Docker Compose Orchestration
yamlservices:
  game-server:
    build: .
    ports:
      - "8080:8080"
    command: ["node", "--require", "source-map-support/register", "dist/src/server.js", "8080", "boards/perfect.txt"]
Deployment Commands:
powershell# Local development (Windows PowerShell)
npm start

# Docker deployment
docker compose up
Show Image

Quality Assurance
Automated Testing Strategy
The test suite covers all game rules (1-A through 3-B) plus edge cases:
Running tests on Windows:
powershellnpm run test
Test Coverage:

✅ Board parsing and validation
✅ Rule 1 (First card): All paths (1-A, 1-B, 1-C, 1-D)
✅ Rule 2 (Second card): All paths (2-A, 2-B, 2-C, 2-D, 2-E)
✅ Rule 3 (Cleanup): Matched pair removal (3-A) and flip-down (3-B)
✅ Concurrent waiting (Rule 1-D deadlock prevention)
✅ Map function transformation
✅ Watch functionality with multiple observers

Show Image
Total Test Count: 25 passing
Code Documentation
I generated API documentation using TypeDoc:
powershellnpm run doc
The documentation includes:

Class hierarchies and inheritance
Method signatures with parameter types
Return value specifications
JSDoc comments explaining functionality

Show Image
Show Image
Show Image

Game Mechanics Verification
Simulation: Automated Players
I implemented two simulation modes to test concurrent gameplay:
Mode 1: Polling with Watcher
powershellnpm run simulation
This mode runs 4 concurrent players making random moves while a dedicated watcher monitors all board changes:
Show Image
Show Image
Observations:

Players correctly wait when trying to flip controlled cards (Rule 1-D)
Matched pairs are removed on next first card flip (Rule 3-A)
Non-matching cards flip down at appropriate times (Rule 3-B)
Watcher detects every board state change

Mode 2: All Players Watch
powershellnpm run simulation watch
In this mode, all 4 players both play AND watch for changes:
Show Image
Key Insight: Every player successfully detects changes made by others, proving the changeResolvers mechanism works correctly for multiple simultaneous watchers.

Game Mechanics Verification (Continued)
Rule 1: First Card Flip Scenarios
Rule 1-A: Empty Space → Failure
Test: Attempting to flip a card after it was removed by a matching pair.
Expected Behavior: Operation fails with "empty space" error.
Result:
Show Image
Code Path:
typescriptif (pic === null) {
    throw new Error('empty space');  // Rule 1-A
}

Rule 1-B: Face-Down Card → Flip Up & Control
Test: Clicking a face-down card as first card.
Expected Behavior: Card flips face-up, player gains yellow control indicator.
Result (My View):
Show Image
Result (Other Player's View):
Show Image
Observation: Both players see the card face-up, but only I see the yellow control indicator (correct behavior per spec).

Rule 1-C: Face-Up Uncontrolled → Take Control
Test Scenario:

Previous player had mismatch (Rule 2-E) → cards left face-up but uncontrolled
I click one of those face-up cards as my first card

Expected Behavior: No visual change (already face-up), but I gain control (yellow).
Before (face-up, no control):
Show Image
After (face-up, with control):
Show Image
Code Implementation:
typescriptif (ctrl === null) {
    controllerRow[col] = playerId;   // Take control without flipping
    player.setFirstCard({ row, col });
}

Rule 1-D: Controlled by Another → Wait
Test: Attempting to flip a card currently controlled by another player.
Expected Behavior: My card turns green (waiting state), operation blocks until released.
Visual Indicator:
Show Image
What's Happening Behind the Scenes:
typescript// My code enters waiting state
const deferred = new Deferred<void>();
waitingForControl.get(key).push(deferred);
await deferred.promise;  // ⏸️ BLOCKS HERE

// When other player releases the card:
notifyWaiters(row, col);  // ▶️ Wakes me up

Rule 2: Second Card Flip Scenarios
Rule 2-A: Second Card is Empty → Fail & Relinquish First
Test Scenario:

I control first card (yellow)
I try to flip an empty space as second card

Expected Behavior:

Error popup
First card loses control (yellow → white) but stays face-up
Will flip down on my next first card flip (Rule 3-B)

Before (first card controlled):
Show Image
After (first card relinquished):
Show Image
Implementation Detail:
typescriptif (pic === null) {
    // Relinquish first card
    firstCtrlRow[firstCard.col] = null;
    this.notifyWaiters(firstCard.row, firstCard.col);  // Wake waiting players
    this.rememberLingering(playerId, firstCard.row, firstCard.col);  // For 3-B
    throw new Error('empty space');
}

Rule 2-B: Second Card Controlled → Fail (Deadlock Avoidance)
Test Scenario:

Other player controls a card
I control a different card as first
I try to flip the other player's card as my second

Expected Behavior:

Immediate failure (no waiting to avoid deadlock)
My first card relinquished
Error message about controlled card

First card controlled:
Show Image
After attempting second card:
Show Image
Show Image
Why No Waiting? If both players waited for each other's cards, we'd have deadlock. The spec requires immediate failure here.

Rule 2-C: Second Card Face-Down → Flip Up
Test: Flipping a face-down card as second card.
Expected Behavior: Card flips up, control assigned.
Show Image

Rule 2-D: Match → Keep Control
Test: Flipping two identical cards in succession.
Expected Behavior: Both cards stay face-up with yellow control until next first card flip (Rule 3-A).
Show Image
State After Match:

Both cards: faceUp = true, controller = playerId
Will be removed on next flipUp() call for this player


Rule 2-E: No Match → Relinquish Both
Test: Flipping two different cards.
Expected Behavior:

Both cards stay face-up but lose control
Will flip down on next first card flip (Rule 3-B)

First card (rainbow):
Show Image
Second card (unicorn) - no match:
Show Image
Result:
Both cards visible but no longer controlled (no yellow highlighting).

Rule 3: Cleanup Before Next First Card
Rule 3-A: Remove Matched Pairs
Test Scenario:

I match two cards (Rule 2-D)
I flip a new first card

Expected Behavior: Previous matched pair disappears from board.
Before (matched pair still visible):
Show Image
After flipping next first card (matched pair removed):
Show Image
Code Execution:
typescriptasync function cleanupPreviousPlay(player: Player): Promise<void> {
    if (firstPic === secondPic && firstPic !== null) {
        // 3-A: Remove matched cards
        firstCardsRow[firstCard.col] = null;
        secondCardsRow[secondCard.col] = null;
        // ... also flip down and release control
    }
}

Rule 3-B: Flip Down Non-Matching Cards
Test Scenario:

I have a mismatch (Rule 2-E) → cards face-up, uncontrolled
I flip a new first card

Expected Behavior: Previous non-matching cards flip face-down.
Before (mismatched cards still face-up):
Show Image
After flipping next first card (mismatched cards flipped down):
Show Image
Conditional Flip-Down:
typescriptprivate flipDownIfUncontrolled(row: number, col: number): void {
    // Only flip down if: exists, face-up, AND uncontrolled
    if (pic !== null && isFaceUp && ctrl === null) {
        faceUpRow[col] = false;
    }
}
This prevents flipping down cards that other players have taken control of in the meantime.

Advanced Features
Problem 4: Map Function - Card Transformation
The map function allows batch transformation of all cards on the board while preserving game state.
Implementation:
typescriptpublic async map(f: (card: string) => Promise<string>): Promise<void> {
    for (let r = 0; r < this.rows; r++) {
        for (let c = 0; c < this.cols; c++) {
            const pic = cardsRow[c];
            if (pic !== null) {
                cardsRow[c] = await f(pic);  // Transform
            }
        }
    }
    this.notifyChange();  // Alert all watchers
}
Test: Replace all unicorn emojis with lollipop emojis
Before transformation:
Show Image
After transformation:
Show Image
Server Endpoint:
typescriptapp.get('/replace/:playerId/:fromCard/:toCard', async(request, response) => {
    const { fromCard, toCard } = request.params;
    await map(board, playerId, async (card: string) => 
        card === fromCard ? toCard : card
    );
});
Key Feature: Face-up/face-down state and control ownership are preserved during transformation.

Problem 5: Watch for Changes
The watch mechanism provides real-time board updates without polling.
Implementation Strategy:
typescript// Server endpoint
app.get('/watch/:playerId', async(request, response) => {
    const boardState = await watch(board, playerId);
    response.status(200).type('text').send(boardState);
});

// commands.ts
export async function watch(board: Board, playerId: string): Promise<string> {
    const { promise, resolve } = Promise.withResolvers<string>();
    board.addChangeWatcher(playerId, resolve);
    return promise;  // Blocks until board changes
}

// board.ts - called after any mutation
private notifyChange(): void {
    for (const [playerId, resolvers] of this.changeResolvers.entries()) {
        const state = this.render(playerId);
        for (const resolve of resolvers) {
            resolve(state);  // Wake up watcher
        }
    }
    this.changeResolvers.clear();
}
Client-Side (Browser):
javascriptfunction watch() {
    const req = new XMLHttpRequest();
    req.addEventListener('load', function onWatchLoad() {
        refreshBoard(this.responseText);  // Update UI
        setTimeout(watch, 1);  // Re-register immediately
    });
    req.open('GET', 'http://' + server + '/watch/' + playerID);
    req.send();
}
Advantages over Polling:

✅ Instant updates (no 2-second delay)
✅ Lower server load (no repeated requests)
✅ Scalable to many watchers (O(1) per change)


Technical Challenges & Solutions
Challenge 1: Deadlock Prevention (Rule 2-B)
Problem: Two players each control one card and try to flip the other's card as their second.
Solution: Rule 2-B explicitly forbids waiting for controlled cards as second cards. One player fails immediately, breaking the potential deadlock.
Code:
typescriptif (isFaceUp && ctrl !== null) {
    // NO WAITING - fail immediately
    throw new Error('card is controlled by another player');
}

Challenge 2: Deferred Cleanup (Rules 3-A/3-B)
Problem: When do we remove matched pairs or flip down mismatched cards?
Solution: The "lingering" mechanism remembers cleanup tasks and executes them before the next first card flip.
Code:
typescriptprivate readonly lingering: Map<string, Array<{row, col}>>;

// When 2-A or 2-B fails:
this.rememberLingering(playerId, firstCard.row, firstCard.col);

// Before processing next first card:
const linger = this.lingering.get(pid);
if (linger) {
    for (const {row, col} of linger) {
        this.flipDownIfUncontrolled(row, col);
    }
}

Challenge 3: Multiple Concurrent Watchers
Problem: How to notify multiple players when the board changes?
Solution: Store an array of resolver functions per player, call all of them on change.
Code:
typescriptprivate readonly changeResolvers: Map<string, ((value: string) => void)[]>;

public addChangeWatcher(playerId: string, resolver: (value: string) => void): void {
    let list = this.changeResolvers.get(playerId) ?? [];
    list.push(resolver);
    this.changeResolvers.set(playerId, list);
}

Performance Characteristics
OperationTime ComplexitySpace ComplexityflipUp() (no waiting)O(1)O(1)flipUp() (with waiting)O(W) where W = waitersO(W)render()O(R × C)O(R × C)map()O(R × C)O(1)notifyChange()O(P) where P = watchersO(1)
Board Dimensions: R = rows, C = columns
Typical Board Size: 3×3 to 5×5 (9-25 cards)

Lessons Learned

Async/Await Simplifies Concurrency: Using promises instead of callbacks made the waiting logic much clearer.
Deferred Cleanup is Elegant: Instead of immediately modifying state during second card flip, deferring cleanup to the next first card flip simplified the logic significantly.
Testing Concurrent Code is Hard: The simulation mode was essential for catching race conditions that unit tests missed.
Source Maps are Critical: Without source-map-support, debugging TypeScript errors would have been painful.
Docker Ensures Consistency: What works on my Windows machine now works identically in the container.


Conclusion
This laboratory successfully implements all five problems:
✅ Problem 1: Board ADT with full concurrency support
✅ Problem 2: HTTP server with REST endpoints
✅ Problem 3: Asynchronous operations with waiting (Rule 1-D)
✅ Problem 4: Map function for card transformation
✅ Problem 5: Watch mechanism for real-time updates
The implementation handles all game rules (1-A through 3-B), prevents deadlocks, and provides a responsive browser-based UI. The code is thoroughly tested, well-documented, and containerized for easy deployment.
Total Development Time: ~15 hours
Final Line Count: ~1200 lines of TypeScript
Test Coverage: 25 tests, all passing
