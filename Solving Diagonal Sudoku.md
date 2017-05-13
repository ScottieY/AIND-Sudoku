# Solving Diagonal Sudoku

## Definition

- **Unit**: the square that can be assigned a value to.(ie. U1 shown below)

- **Box**:  3 * 3 square shown as below.

  ```
  +---+---+---+---+---+---+---+---+---+
  | U1|   |   |           |           |
  +---+---+---+           |           |
  |   |   |   |   Box 2   |  Box 3    |
  +---+---+---+           |           |
  |   |   |   |           |           |
  +---+---+---+-----------+-----------+
  |           |           |           |
  |           |           |           |
  |   Box 4   |   Box 6   |  Box 6    |
  |           |           |           |
  |           |           |           |
  +-----------+-----------+-----------+
  |           |           |           |
  |           |           |           |
  |   Box 7   |   Box 8   |  Box 9    |
  |           |           |           |
  |           |           |           |
  +-----------+-----------+-----------+
  ```

  ​

- **Peer**: a **row, colomn, diagonal or box** that contains the unit

## Techiniques 

### Strategy

#### Elimination####

Eliminate impossible numbers for each unit.

Before **Elimination**:

![alt text][before_elimination]

After **Elimination**:

![alt text][after_elimination]

#### Only Choice

If the unit would only allow a certain digit, then the unit must be assigned to that value

![alt text][only_choice]

#### Search

When there are multiple choices for one unit, try out different values and see which one works. Start with the unit with least possibilities. Using **Deep First Search** algorithm.

**Deep First Search** exampe from [Google's AlphaGo paper][paper].

![alt text][dfs]

### Constraint Propgation

Reducing the puzzle by applying the strategies (constraints) to reduce the possible numbers for each unit.

In this case apply **[Elimination](#Elimination), [Naked Twin](#Naked-Twin) and [Only Choice](#Only-Choice)** one by one.

### Advanced Strategy

#### Naked Twin

```
Before:
+------+-----+-----+        
|  39  |  8	 |	7  |
+------+-----+-----+
|  6   |  5  |	4  |
+------+-----+-----+
|  23  |  23 |	1  |
+------+-----+-----+
After:
+------+-----+-----+
|  9   |  8	 |	7  |
+------+-----+-----+
|  6   |  5  |	4  |
+------+-----+-----+
|  23  |  23 |	1  |
+------+-----+-----+

The 3 can be eliminated from the 39 cell (top left)
```
More example:

Before apply **Naked Twin**

![alt text][before_naked_twin]

After apply **Naked Twin**

![alt text][after_naked_twin]

## Solution

```python
#initialization
def cross(a, b):
    return [s+t for s in a for t in b]

assignments = []
rows = 'ABCDEFGHI'
cols = '123456789'
boxes = cross(rows, cols)

row_units = [cross(r, cols) for r in rows]
column_units = [cross(rows, c) for c in cols]
square_units = [cross(rs, cs) for rs in ('ABC','DEF','GHI') for cs in ('123','456','789')]
diaganol_units = [[s+t for s,t in  zip(rows,cols)],[s+t for s,t in zip(rows,cols[::-1])]]

unitlist = row_units + column_units + square_units + diaganol_units
units = dict((s, [u for u in unitlist if s in u]) for s in boxes)
peers = dict((s, set(sum(units[s],[]))-set([s])) for s in boxes)


def assign_value(values, box, value):
    """
    Please use this function to update your values dictionary!
    Assigns a value to a given box. If it updates the board record it.
    Used for visualization
    """
    values[box] = value
    if len(value) == 1:
        assignments.append(values.copy())
    return values

def display(values):
    """
    Display the values as a 2-D grid.
    Input: The sudoku in dictionary form
    Output: None
    """
    width = 1+max(len(values[s]) for s in boxes)
    line = '+'.join(['-'*(width*3)]*3)
    for r in rows:
        print(''.join(values[r+c].center(width)+('|' if c in '36' else '')
                      for c in cols))
        if r in 'CF': print(line)
    return

def grid_values(grid):
    """Convert grid string into {<box>: <value>} dict with '123456789' value for empties.

    Args:
        grid: Sudoku grid in string form, 81 characters long
    Returns:
        Sudoku grid in dictionary form:
        - keys: Box labels, e.g. 'A1'
        - values: Value in corresponding box, e.g. '8', or '123456789' if it is empty.
    """
    dict = {}
    for idx,val in enumerate(grid):
        key = chr(int(idx / 9) + ord('A')) + chr(int(idx % 9) + ord('1'))
        if val == '.':
            dict[key] = '123456789'
        else:
            dict[key] = val
        
    return dict
    
def eliminate(values):
    """
    Go through all the boxes, and whenever there is a box with a value, eliminate this value from the values of all its peers.
    Input: A sudoku in dictionary form.
    Output: The resulting sudoku in dictionary form.
    """
    solved_values = [box for box in values.keys() if len(values[box]) == 1]
    for box in solved_values:
        digit = values[box]
        for peer in peers[box]:
            if digit in values[peer]:
                values = assign_value(values,peer,values[peer].replace(digit,''))
    return values

def only_choice(values):
    """
    Go through all the units, and whenever there is a unit with a value that only fits in one box, assign the value to this box.
    Input: A sudoku in dictionary form.
    Output: The resulting sudoku in dictionary form.
    """
    for unit in unitlist:
        for digit in '123456789':
            dplaces = [box for box in unit if digit in values[box]]
            if len(dplaces) == 1 and len(values[dplaces[0]]) != 1:
                values = assign_value(values,dplaces[0],digit)
    return values

def reduce_puzzle(values):
    """
    Iterate eliminate() and only_choice(). If at some point, there is a box with no available values, return False.
    If the sudoku is solved, return the sudoku.
    If after an iteration of both functions, the sudoku remains the same, return the sudoku.
    Input: A sudoku in dictionary form.
    Output: The resulting sudoku in dictionary form.
    """
    solved_values = [box for box in values.keys() if len(values[box]) == 1]
    stalled = False
    while not stalled:
        # Check how many boxes have a determined value
        solved_values_before = len([box for box in values.keys() if len(values[box]) == 1])
        values = naked_twins(values)
        # Use the Eliminate Strategy
        values = eliminate(values)
        # Use the Only Choice Strategy
        values = only_choice(values)
        # Check how many boxes have a determined value, to compare
        solved_values_after = len([box for box in values.keys() if len(values[box]) == 1])
        # If no new values were added, stop the loop.
        stalled = solved_values_before == solved_values_after
        # Sanity check, return False if there is a box with zero available values:
        if len([box for box in values.keys() if len(values[box]) == 0]):
            return False
    return values

def search(values):
    "Using depth-first search and propagation, create a search tree and solve the sudoku."
    # First, reduce the puzzle using the previous function
    remain_puzzle = reduce_puzzle(values)
    # Choose one of the unfilled squares with the fewest possibilities
    if remain_puzzle is False:
        #Current puzzle is unsolvable
        return False
    if all(len(remain_puzzle[s]) == 1 for s in boxes):
        #Current puzzle is solved
        return remain_puzzle
    min_key = None
    #Find the unit with least possible numbers
    for key,val in remain_puzzle.items():
        if (len(val) != 1):
            if (min_key == None) or (len(val) < len(remain_puzzle[min_key])):
                min_key = key
    #Try the one of the possibility with the unit found previously
    for char in remain_puzzle[min_key]:
        new_puzzle = remain_puzzle.copy()
        new_puzzle[min_key] = char
        solution = search(new_puzzle)
        if solution:
            return solution

def make_pair(a,b):
    """
    Helper function for naked_twins
    Create ordered pair of units
    Input:
        a: Unit 
        b: Unit
    Output:
        Ordered pair(tuple)
    Example:
        make_pair("A1","D1") == makepair("D1","A1") == ("A1","D1")
    """
    if  a>b:
        return (a,b)
    else:
        return (b,a)

def naked_twins(values):
    unsolved = [box for box in boxes if len(values[box]) != 1]

    pairs = set([])
    #Find twins
    for box in [b for b in unsolved if len(values[b]) == 2]:
        for peer in [p for p in peers[box] if values[p] == values[box]]:
            pairs.add(make_pair(box,peer))

    #Eliminate twins
    for a,b in pairs:
        for unit in [u for u in units[a] if b in u]:
            for box in [bx for bx in unit if len(values[bx]) > 1 and bx != a and bx != b]:
                for char in values[b]:
                    values = assign_value(values,box,values[box].replace(char,''))

    return values

def solve(grid):
    """
    Find the solution to a Sudoku grid.
    Args:
        grid(string): a string representing a sudoku grid.
            Example: '2.............62....1....7...6..8...3...9...7...6..4...4....8....52.............3'
    Returns:
        The dictionary representation of the final sudoku grid. False if no solution exists.
    """
    values = grid_values(grid)
    result = search(values)
    return result
```





[before_elimination]:https://d17h27t6h515a5.cloudfront.net/topher/2017/January/586dc35f_reduce-values/reduce-values.png
[after_elimination]: https://d17h27t6h515a5.cloudfront.net/topher/2017/January/586dd40f_values-easy/values-easy.png
[only_choice]:https://d17h27t6h515a5.cloudfront.net/topher/2017/January/5887d815_only-choice/only-choice.png
[paper]:https://storage.googleapis.com/deepmind-media/alphago/AlphaGoNaturePaper.pdf
[dfs]:https://d17h27t6h515a5.cloudfront.net/topher/2017/January/5885ac6f_screen-shot-2017-01-22-at-11.08.01-pm-1/screen-shot-2017-01-22-at-11.08.01-pm-1.png
[before_naked_twin]:https://d17h27t6h515a5.cloudfront.net/topher/2017/January/5877cc63_naked-twins/naked-twins.png
[after_naked_twin]:https://d17h27t6h515a5.cloudfront.net/topher/2017/January/5877cc78_naked-twins-2/naked-twins-2.png
[]:

