# LLD Foundations: General Software Design Principles

*The principles that sit above SOLID — HelloInterview treats these as the ones to internalize first, since SOLID only kicks in once you're already designing class hierarchies.*

---

## 0. Why These Come Before SOLID

SOLID applies specifically to object-oriented class design. These five are broader — they apply to any code, in any paradigm, and they're the ones that catch the most common interview mistakes. If you only remember three principles from this whole file, make it **KISS, DRY, and YAGNI** — those three alone will carry you through most interviews.

---

## 1. KISS — Keep It Simple, Stupid

**The simplest solution that works is usually the right one.**

If a plain conditional solves the problem, use a conditional — don't reach for a Strategy pattern to look sophisticated. If a single class handles the job cleanly, don't split it into three just because "separation" sounds good on paper.

**This is the single most commonly violated principle in LLD interviews.** Candidates over-engineer because they want to demonstrate pattern knowledge — introducing factories, builders, and decorators where a basic class would work fine. Interviewers notice this immediately, and it reads as a lack of judgment rather than depth of knowledge. What they actually want to see is that you can tell the difference between a problem that needs a sophisticated solution and one that doesn't.

**When to add complexity:** only once simplicity actually stops working. If a single class balloons to 500 lines with ten responsibilities, that's your refactor signal. If adding a new payment method means editing code in five different places, that's your signal to introduce a Strategy pattern. Complexity is earned, not front-loaded.

**Interview tell:** if you're about to introduce a pattern, ask yourself "am I doing this because the problem needs it, or because I want to show I know it?" Only the first reason is a good one.

---

## 2. DRY — Don't Repeat Yourself

**When the same logic shows up in multiple places, pull it into one place.**

If three classes all validate an email address the same way, extract a shared validation method. If two services both convert timestamps the same way, put that conversion in one utility function.

**The payoff is maintenance cost.** When validation rules change, you update one method instead of hunting through the codebase for every duplicate. When a bug exists in the timestamp conversion, you fix it exactly once.

**Where DRY goes too far:** if two pieces of code merely *look* similar but serve genuinely different purposes, duplication can be the right call. Forcing them to share an abstraction creates artificial coupling — a change to satisfy one use case silently breaks the other. The real test isn't textual similarity, it's whether the logic is *conceptually* the same thing.

**DRY vs. KISS — a real tension, not a contradiction to paper over.** Sometimes the simplest thing is to duplicate a small piece of logic in two places rather than build a shared abstraction too early. There's no universally right answer, and being able to articulate the tradeoff live is what tends to separate senior candidates from mid-level ones. A strong thing to say out loud in an interview:

> "I'd expect this validation logic to show up in more than one place eventually, but I'll start by keeping it local to the `User` class to avoid premature abstraction. If it shows up three or four times, that's my signal to extract a shared validator."

That single sentence demonstrates you can hold two competing principles at once instead of blindly following either one.

---

## 3. YAGNI — You Aren't Gonna Need It

**Build what the requirements ask for now — not what you imagine you might need later.**

If you're designing a parking lot system and the requirements don't mention valet parking or EV charging, don't add support for them. Don't make every class maximally extensible "just in case."

**Why speculative extensibility backfires:** you usually guess wrong about what the future requirement will actually look like. You pay the complexity cost for scenarios that never happen, and when a real new requirement does show up, it rarely matches what you pre-built for — so you end up maintaining dead code *and* still doing the real work from scratch.

**Important nuance:** YAGNI doesn't mean "never think ahead." It means don't *build* ahead. It's fine — good, even — to design with extension points in mind (an interface here, a seam there), as long as you only *implement* what's actually needed right now.

**Where this shows up explicitly in interviews:** when the interviewer asks "how would you extend this to support X?" — that's your cue to talk through how the design *would* change. But your initial design itself should stick strictly to what's been asked for. Building the extension unprompted, before it's asked for, is the YAGNI violation.

**Cross-reference:** this is exactly why `parking-lot-notes.md`'s "out of scope" list (reservations, full payments, valet, EV charging) exists as an explicit section — writing it down keeps you honest about what you're not building, and gives you a ready answer when the interviewer does ask about extensibility later.

---

## 4. Separation of Concerns

**Different parts of your code should own different responsibilities, and shouldn't reach into each other's internals.** A UI layer shouldn't contain business logic. Business logic shouldn't know how data gets persisted. A data-access layer shouldn't be formatting strings for display.

### Code smell
```java
// BAD: display logic, input handling, and game rules all tangled into one method
class TicTacToe {
    char[][] board = new char[3][3];

    void play() {
        Scanner scanner = new Scanner(System.in);
        while (true) {
            for (char[] row : board) System.out.println(row);   // display
            int row = scanner.nextInt();
            int col = scanner.nextInt();                          // input handling
            board[row][col] = 'X';
            if (board[0][0] == board[1][1] && board[1][1] == board[2][2]) { // win check
                System.out.println("Winner!");
                break;
            }
        }
    }
}
```

### Fix
```java
// GOOD: each concern lives in its own class
class TicTacToe {
    Board board;
    Display display;
    InputHandler input;

    void play() {
        while (!board.hasWinner()) {
            display.render(board);
            Move move = input.getNextMove();
            board.makeMove(move);
        }
        display.showWinner(board.getWinner());
    }
}
```

Now switching from console input to a GUI only touches `InputHandler`. Changing how the board renders only touches `Display`. Adding a new win condition only touches `Board`. Each change is isolated to exactly one class — and each piece can be tested independently of the others.

**Relationship to SRP:** Separation of Concerns is the general-software-design version of what SRP does specifically for classes. If you're already applying SRP well, you're usually already respecting this.

---

## 5. Law of Demeter (Principle of Least Knowledge)

**A method should only talk to its immediate collaborators, not reach through them to grab something three objects deep.**

If you see code like `order.getCustomer().getAddress().getZipCode()`, that's a violation — your code now depends on the internal structure of three different objects it has no business knowing about.

**Why deep chaining is a problem:** it's a coupling issue. If any of those three objects reorganizes its internal data, your calling code breaks — even though conceptually you only ever wanted one piece of information (the zip code). The fix is to put a method directly on `Order`, e.g. `getCustomerZipCode()`, that handles the internal navigation for you.

**Important distinction — this is *not* the same as method chaining being bad.** Fluent interfaces like `builder.setName("John").setAge(30).build()` are fine, because each call returns the *same* object type — you're not tunneling through unrelated objects, you're just calling methods sequentially on one thing. The violation is specifically about traversing through *multiple different object types* to reach data several layers removed from where you started.

**Where this shows up in interviews:** when you're defining method signatures on your classes. Instead of returning a complex object that forces the caller to dig through it, either return the specific piece of data the caller actually needs, or provide a higher-level method that does the navigation internally.

---

## 6. Putting It All Together

You don't need to name-drop these principles constantly in an interview. Use them to *guide* your decisions, and reference one briefly when explaining a tradeoff — they're thinking tools, not a checklist to recite out loud.

| Principle | One-line summary |
|---|---|
| KISS | Start simple; add complexity only once simplicity genuinely stops working |
| DRY | Reduce duplication — but only when the logic is conceptually the same, not just textually similar |
| YAGNI | Build for the stated requirements, not imagined future ones |
| Separation of Concerns | Each part of the system owns one concern and doesn't reach into another's internals |
| Law of Demeter | Talk to immediate collaborators only; don't tunnel through object chains |

---

## 7. Suggested Resources

- HelloInterview — [Design Principles](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/design-principles) — source for the framing and examples above; also where SOLID lives on the same page, positioned as the *object-oriented-specific* subset of these broader principles.
- Your own codebase — the best way to internalize KISS/YAGNI specifically is to look back at a class you wrote for a hypothetical future need that never actually arrived, and ask honestly whether it was worth the complexity.

---

## 8. Self-Test

1. An interviewer asks you to design a notification system. You add support for Email, SMS, *and* Push notifications, even though only Email was mentioned in requirements. Which principle did you violate, and what should you have done instead?
2. You've written the same "is this string a valid US phone number" check in three different validator classes with slightly different formatting. Should you DRY this up immediately? What would you ask yourself first?
3. Give an example of method chaining that is **not** a Law of Demeter violation, and explain why it's different from `a.getB().getC().getD()`.
4. Why is Separation of Concerns considered the general-software-design counterpart to SRP specifically?
5. You catch yourself about to introduce a Factory pattern for a class that only ever has one implementation. What question should you ask before doing it?
