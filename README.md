# Superball

These are the interfaces for the classes I used for this solution, I omitted all the constructors except for `BoardAnalyzer` for simplicity. Most of these are probably not important, read through the code below, and come back up here for reference if you need it.

```cpp
typedef std::unordered_map<char, size_t> ColorValues;

// ## Board structs

struct Space {
  char character; // this needs to be mutable for swaps

  const size_t row;
  const size_t column;
  const size_t index;
  const bool goal;

  bool is_empty() const;
};

struct Board {
  std::vector<Space> spaces;

  size_t empty_spaces = 0;

  const size_t rows;
  const size_t columns;
  const size_t size;

  const size_t minimum_score_size;
  const ColorValues color_values;

  void swap_spaces(Space &space1, Space &space2);
};

struct Superball {
  Board board;

  void analyze() const;
  void play();
};

// ## Analysis structs, these are also what I used in `sb-analyze`

// scrape the information from the disjoint set and store it in a map for easy
// access and iteration
struct Set {
  const size_t id;
  const size_t size;

  const bool has_goal_space;
  const bool scorable;
  const size_t points;

  struct {
    const char character;
    const size_t row;
    const size_t column;
  } goal;

  void SCORE() const {
    printf("SCORE %zu %zu\n", goal.row, goal.column);
    exit(0);
  }
};

struct BoardAnalyzer {
  std::unordered_map<size_t, Set> sets;

  BoardAnalyzer(const Board &board); // Load in the Superball.board member

  void try_to_score() const;
  size_t get_score() const;
};
```

To develop a strategy, I played the game for an hour and just tried to get an idea of how it worked. I noticed, since five tiles appeared every turn, you needed to clear as many tiles as possible when scoring a set in order to keep up. Once I got a feel for that, I went off a few basic assumptions:

1. Always wait until the last possible moment to score a set, don't try to score unless the board will fill up on the next turn. This will maximize the value you get from swaps.
2. When you make swaps, try to make swaps that will increase the size of two sets at once.
3. Add a higher priority to sets with a goal space.

I did not worry about the point values of the sets, I mostly just focused on clearing tiles and staying alive. I figured the larger scoring sets would clear eventually.

Overall, I still think this was a pretty basic solution. I came up with a simple way of doing this, and I was surprised at how well it worked honestly. It worked well enough that I didn't spend a lot of time coming up with elaborate solutions, so I think what I did probably still could have been refined plenty more.

In a nutshell, I tried each, swap, computed a score for each swap, then pick the best score. This was by no means an efficient solution.

Here's the code for the scoring function: all I did was square each set, this would catch swaps that increase the size of two sets at once, and it would also prioritize larger sets so the most tiles could be cleared out in a single turn. Then I cubed sets with a goal space in them to give them higher priority.

```cpp
size_t BoardAnalyzer::get_score() const {
 size_t score = 0;
 for (const auto &it : sets) {
   score += pow(it.second.size, it.second.has_goal_space ? 3 : 2);
 }
 return score;
}
```

Here's the code for my `sb-play` file

```cpp
#include <vector>

#define SPACES_FILLED_PER_TURN 5

int main(const int argc, const char **argv) {
  Superball(argc, argv).play();
  return 0;
}

// Handles printing my choice
struct Swap {
  const Space &space1;
  const Space &space2;
  Swap(const Space &space1, const Space &space2)
      : space1(space1), space2(space2) {}
  void SWAP() {
    printf("SWAP %zu %zu %zu %zu\n", space1.row, space1.column, space2.row,
           space2.column);
    exit(0);
  }
};


void Superball::play() {
  // Assumption 1, don't score unless the board will fill up on the next turn
  if (board.empty_spaces < SPACES_FILLED_PER_TURN) {
    BoardAnalyzer(board).try_to_score(); // Goes through all the goal spaces and tries to call the `SCORE` function on the best scoring set. If none exists, game over.
  }

  struct {
    size_t score = 0; // best score
    std::vector<Swap> swaps; // list of best swaps (could possibly be more than
                             // one, didn't really test this though)
  } best;

  for (size_t i = 0; i < board.size - 1; ++i) {
    Space &space1 = board.spaces[i];

    if (space1.is_empty()) {
      continue;
    }

    for (size_t j = i + 1; j < board.size; ++j) {
      Space &space2 = board.spaces[j];

      if (space2.is_empty()) {
        continue;
      }

      // If they're the same color, keep going
      if (space1.character == space2.character) {
        continue;
      }

      board.swap_spaces(space1, space2);

      const size_t score = BoardAnalyzer(board).get_score();

      // set new best
      if (score > best.score) {
        best.score = score;
        best.swaps.clear(); // reset the list of swaps
      }
      // push back the new set and any subsequent duplicates
      if (best.score == score) {
        best.swaps.push_back(Swap(space1, space2));
      }

      // un-swap the spaces
      board.swap_spaces(space2, space1);
    }
  }

  // I thought about adding code here to handle ties (not even sure if there were many), but I just went with the first swap
  best.swaps.front().SWAP();
}
```

Overall my solution did well on certain ones, I think I had one crazy one that scored like 30,000 or something, but for others, it scored only a few hundred.
