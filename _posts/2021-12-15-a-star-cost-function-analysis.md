---
toc: true
layout: post
description: Some experimentation on different A* cost functions 
categories: [algorithms]
title: A* Cost function experimentation
---

# A* Cost Function Experimentation

## Introduction

For several years, I have participated in the Advent Of Code, a series of challenges in the month of december. This year, on day 15, the challenge required navigating a grid where each step had a certain costs between 0 and 9. It allowed me to try to solve the challenge using the A*-algorithm, a variation of Dijkstra's Shortest Path Algorithm. 

Part of the A*-algorithm, where it deviates from Dijkstra's is in determining a cost function where an estimation is made of the cost from a specific position to the end. A good cost function has the effect of "aiming" the algorithm more towards the goal, so it will no longer inspect all possible paths, but will prioritize paths that are closer and closer to the goal. This gives A* the benefit of being faster than Dijkstra in big search spaces (because you no longer have to inspect all possible options), but it might not give the exact shortest path, since the most optimal path might require a bit of a detour.

Using this challenge, I have tried some experimentation on the cost function to see the effect on the algorithm, and tried to make the effect.

![]({{site.baseurl}}/images/diagram.png)

## The challenge

The challenge can be found on [Advent of Code 2021, day 15](https://adventofcode.com/2021/day/15). The gist is that a grid is provided with numbers that represent the "cost" of moving into that particular square. For example:

```
1163751742
1381373672
2136511328
3694931569
7463417111
1319128137
1359912421
3125421639
1293138521
2311944581
```

The actual puzzel input you get is bigger, and is a grid of 100x100 numbers.
Now, the challenge is to find the shortest path (i.e. path with the least cost total) from the top-left to the bottom-right.

## The A* Implementation


To get off to a flying start, below is my implementation of the A* algorithm. First I defined a new `Node` class to represent each square:

```C#
public class Node
{
    public Node Parent { get; set; }
    public int X { get; set; }
    public int Y { get; set; }

    public int F => this.G + this.H;
    public int G { get; set; }
    public int H { get; set; }

    public int Value { get; set; }
}
```

Each square in the grid is a `Node`, keeping track of its location, the values `G`, `H`and `F`) that are important in the A* algorithm, its parent (Which can be used in the end to backtrack through the path that was found) and its value, which is just the cost of the square (the number in the puzzel input above)

The full solution has  A* steps  annotated in the code below. There are some rendering calls that just render a visual reprentation on the screen, and a `Backtrack()` function that loops through the path and adds up the costs. These can be ignored for this example. 

```C#
public async Task Part1()
{
	Node[][] grid = new Node[this.Input.Count()][];

    // Parse the input to a 2d array of Nodes
	for (int row = 0; row < this.Input.Count(); row++)
    {
    	string line = this.Input.ElementAt(row);
        grid[row] = line.Select((x, i) => new Node
        {
        	X = i,
            Y = row,
            Value = int.Parse(x.ToString())

		}).ToArray();
    }

	// A*

	// Init open/closed lists
	IList<Node> open = new List<Node>();
	IList<Node> closed = new List<Node>();

	// Define start and end nodes
	(int x, int y) start = (0, 0);
	(int x, int y) end = (grid[0].GetLength(0) - 1, grid.GetLength(0) - 1);

	// Add starting node to open
	open.Add(grid.SelectMany(g => g).Single(n => n.X == start.x && n.Y == start.y));

	RenderGrid(this.Input);

	// While open is not empty
	while (open.Any())
	{
		// Get the current node with lowest F
		Node current = open.OrderBy(x => x.F).First();
		RenderCurrent(current);

		// remove the currentNode from the openList
		// add the currentNode to the closedList
		open.Remove(current);
		closed.Add(current);

		// Check for goal
		if (current.X == end.x && current.Y == end.y)
		{
			// Current node is the goal node. Backtrack through parants to find the path
			RenderGrid(this.Input);
			RenderPath(current);

			int cost = Backtrack(grid, current);
			answers.WriteLine($"Answer Part 1: {cost}");
			break;
		}

		// Get children of current node
		IList<Node> children = Get4Children(grid, current);
		foreach (var child in children)
		{
			if (closed.Contains(child, new NodeComparer()))
			{
				// Node already visited, do nothing
				continue;
			}

			if (open.Contains(child, new NodeComparer()))
			{
				foreach (var openNode in open)
				{
					if ((openNode.X == child.X && openNode.Y == child.Y) && child.G < openNode.G)
					{
						// this node has a lower G cost than the current node, set its parent to current
						openNode.Parent = current;
					}
				}
			}
			else
			{
				child.G = current.G + child.Value; // Cost of current to child

				// Cost function, this is a low estimate of current to the end.
			    child.H = 0; //Dijkstra

				child.Parent = current;
				open.Add(child);
			}

		}
	}
}
```

Of note for this post is the calculation of the `H`. This allows us to "steer" the algorithm. Now, lets' experiment with the cost function and see what happens.

## Dijkstra

```
child.H = 0;
``` 
Setting the Cost function to 0, or simple not doing any optimizations, the algorithm will basically execute Dijkstra's algorithm and inspect all nodes. A visual representation looks like this:

![]({{site.baseurl}}/images/a-star-experimentation/dijkstra.gif)

One can see that all nodes are being inspected (painted magenta), and the optimal path is found at the end. Because all nodes are checked. This will guarantee the actual shortest path, since all nodes are checked, but the algorithm also checks nodes that are way out of the way of the path. going from the top-left to the bottom-right, it is probably not needed to check if there is an optimal path that goes through the top-right or bottom-left.

Let's try something else.

## Manhattan-Distance

```
child.H = Math.Abs(end.x - child.X) + Math.Abs(end.y - child.Y);
``` 

One interpretation of the cost from any point to the end is the Manhattan distance. This is basically the number of steps you have to walk down and right to get from any point to the end. The Algorithm now looks like this:

![]({{site.baseurl}}/images/a-star-experimentation/manhattan.gif)

While there seems to be a little more emphasis on the algorithm to prioritize the bottom-right direction, it is not really noticable. One problem might be that the search space is relatively small. It is just a 100x100 box. Maybe we can add a factor to the cost to really emphasize the cost?  

## Manhattan-Distance with a factor

```
child.H = (Math.Abs(end.x - child.X) + Math.Abs(end.y - child.Y)) * factor;
```

Multiplying the cost by a factor maybe helps to offset this small search space and really force the algorithm in the direction we want.

Lets try a factor of 2:

![]({{site.baseurl}}/images/a-star-experimentation/manhattan-2.gif)

That definately looks like something! It completely skips the top-right! We still inspect the bottom-left, but it might just be lower numbers in general down there.

Maybe a factor of 3?

![]({{site.baseurl}}/images/a-star-experimentation/manhattan-3.gif)

That is very pronounced! We really skip a lot of the grid, and really focus on just checking interesting nodes that bring us closed to the end. Interesting are the branches that shoot out when a clump of nodes that looked promising at first suddenly had a higher cost than a new path that was found more in the direction of the goal.

## Pythagoras with a factor

```
child.H = ((int)Math.Sqrt(Math.Pow(end.x - child.X, 2) + Math.Pow(end.y - child.Y, 2))) * factor;
```

Instead of using the Manhattan distance, which is an over-estimation, maybe Pythagoras works better? I have tried this with several factors and below are the results for factor 4:

![]({{site.baseurl}}/images/a-star-experimentation/pythagoras-4.gif)

and factor 5:

![]({{site.baseurl}}/images/a-star-experimentation/pythagoras-5.gif)

Now there is some real force pushing us to the bottom-right. It is so much faster! But did we really found the optimal path?

## Conclusion

Playing with the cost function of the A* is a great way to optimize your pathfinding algorithm. The more powerful the function, the less of the search space needs to be inspected and the faster a result is gotten. However we forego the idea that we really have the most optimal path, the actual shortest path might be one we didn't inspect.

As always, there is a trade-off between speed and correctness, so experiment and find the optimal solution!