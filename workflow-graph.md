
Milchdealer's Blog
BlogHome
Implementing a workflow graph

2020-07-19 • Milchdealer
Goal

Implement a struct which holds the workflow graph and yields the next values in line for execution.
Workflow Graph

When writing a workflow management engine, one has to decide on the data structure of the workflow. I decided to go for the most common one: A directed acyclic graph, in short DAG. I thought about using a more open type, i. e. a cyclic graph, but that has really weird implications for a retry system and output handling. It would be a cool idea to explore, but it sounds like a pain to implement.

I have not yet written anything about OpenWorkflow, but will be using the types here. Most of them should be self-explanatory, and I’m sure there will be posts about OpenWorkflow eventually, but if you’re interested just check out the protobuf definition in the linked repository.
Implementing a DAG

Luckily for me, there is a very extensive graph library in rust, petgraph. So first of I need to create the graph to hold my task instances.

```rust
use chrono::Duration;
use petgraph::Graph;
use openworkflow::{Execution, RunCondition, ExecutionStatus, Task};
#[derive(PartialEq, Clone)]
pub struct TaskInstance {
	task_id: String,
	retries: u32,
	max_retries: u32,
	retry_interval: Duration,
	execution_details: Execution,
	execution_status: Option<ExecutionStatus>,
	run_condition: RunCondition,
	downstream_tasks: Vec<String>,
}

type Node = TaskInstance;
type Edge = RunCondition;
pub struct Dag {
	graph: Graph::<Node, Edge>,
}
```
And then a way to fill my Dag with a list of tasks, parsed from a protobuf message. For this Dag implements the TryFrom trait.

```rust
use std::convert::TryFrom;

impl TryFrom<&Vec<Task>> for Dag {
	type Error = FlowtyError;

	fn try_from(tasks: &Vec<Task>) -> Result<Dag, Self::Error> {
		// deducted \\
	}
```

FlowtyError is using SNAFU to create errors for Flowty.

```rust
use snafu::Snafu;

#[derive(Debug, Snafu)]
pub enum FlowtyError {
	// deducted \\
	#[snafu(display("Failed to parse DAG"))]
	ParsingError,
	#[snafu(display("Cyclic dependency detected!"))]
	CyclicDependencyError,
}
```

Now that that’s out of the way, let’s populate the graph. First we plainly create it and iterate over our list of tasks.

```rust
let mut graph = Graph::<Node, Edge>::new();
for task in tasks {
	if task.execution.is_none() {
		return Err(FlowtyError::ParsingError);
	}
	let retry_interval = Duration::from_std(
		task.retry_interval
		.clone()
		.unwrap_or_default()
		.try_into()
		.unwrap_or_default()
	).unwrap();
	let ti = TaskInstance {
		task_id: task.task_id.clone(),
		retries: 0,
		max_retries: task.retries,
		retry_interval,
		execution_details: task.execution.clone().unwrap(),
		execution_status: None,
		run_condition: RunCondition::from(task.condition),
		downstream_tasks: task.downstream_tasks.clone(),
	};
	graph.add_node(ti);
}
```

Parsing from a prost_types::Duration to a chrono::Duration is a bit of a pain, going over the std::time::Duration type, but it is what it is. Now we have a graph full of nodes, but no edges. To change that we iterate through all node indices and check their downstream dependencies.

```rust
for parent_index in graph.node_indices() {
	let ti = &graph[parent_index].clone();
	for downstream_task in &ti.downstream_tasks {
		match graph.node_indices().find(|i| graph[*i].task_id == *downstream_task) {
			Some(child_index) => {
				graph.update_edge(parent_index, child_index, graph[child_index].run_condition);
			},
			None => ()
		};
	}
}
```

We add an edge of the task_id if a downstream task is matching a node in our graph. Any typos and so on in that list of downstream will, for now, just be ignored. Now we have a directed graph, since that is the petgraph default. Before returning let’s check if it’s acyclic.

```rust
if algo::is_cyclic_directed(&graph) {
	return Err(FlowtyError::CyclicDependencyError);
}
```

Nice! Petgraph comes with a built-in functionality. Now our Dag is complete.

```rust
Ok(Dag{graph})
```

Traversing (or, getting the current execution stage)

So now comes the part you’ve been waiting for: Traversing. Or rather, getting the current execution stage.

You see, it’s not so much that I want to drain my graph when passing nodes, nor that I want to get to the end of it immediatly. Instead, I want to know which nodes (tasks) are executing, or are up for execution. Which is why Dag also implements Iterator. An iterator allows to ask for the next item in line. It’s a bit of an abuse of the functionality, because using this iterator in a for loop will block the entire thead until the graph is completly done. But calling next whenever we want to know if action is required on the Dag is handy.

Let’s start from the top. Namely using a topologic sort (thanks petgraph) to visit each node in the correct order.

```rust
impl Iterator for Dag {
	type Item = Vec<NodeIndex>;

	/// Traverse the Dag via the Iterator.
	/// Returns the current execution stage of the Dag. Meaning all TaskInstances which are currently executing or
	/// ready for execution.
	///
	/// Uses toposort to start from the top, then checks if a task's run_condition is met.
	/// Whenever a task is added to the current stage, it's downstream tasks are saved.
	/// If a task does not appear in the saved downstream list, it's immediatly added to the stage.
	/// If a task is inside the downstream list, its run_condition is checked.
	fn next(&mut self) -> Option<Self::Item> {
		let mut stage: Self::Item = Vec::new();
		let mut downstream: Vec<String> = Vec::new();
		for node in algo::toposort(&self.graph, None).unwrap() {
			if task_instance_is_done(&self.graph[node]) {
				continue;
			}
			// deducted \\
		}
		// deducted \\
	}
```

It’s already explained in the comment above the function, but let’s write it out step-by-step. The toposort is already here, returning the index for each node in order. We also create what we will later return, the stage, and the list of downstream tasks of the tasks in the current stage. Additionally, I added a check at the very beginning of the loop, calling task_instance_is_done() which returns true if the status matches to success or failure. In that case this node is already a-okay✅.

Now we check if the current task is already in the downstream vector. If it is not, we can directly add it to our stage because we get all results from toposort in order.

```rust
let downstream_tasks = &self.graph[node].downstream_tasks;
if downstream.contains(&self.graph[node].task_id) {
	// deducted \\
} else {
	stage.push(node);
	downstream.append(&mut downstream_tasks.clone());
}
```

What’s missing now is the logic to determine whether something should be added to the current stage, even if it is part of the downstream dependencies. So let’s match over the run_condition of our node. The important conditions here are

    None, execute without caring for dependencies
    OneDone, execute when one parent is done
    OneSuccess, execute when one parent has succeeded
    OneFailed, execute when one parent has failed

The reason behind this is, that if all parents were done, it should not have been listed in the downstream vector in the first place. I implemented the logic for all possible cases in the match. Rust match is exhaustive, although I could’ve probably just combined them all with _ => (). But I decided it would be nice to have, and I will most likely extend the features of this code later.

For now, let’s dive into the match, and match to None.

```rust
match self.graph[node].run_condition {
	RunCondition::None => {
		stage.push(node);
		downstream.append(&mut downstream_tasks.clone());
	},
	// deducted \\
}
```

Simple enough, we push the current node to the stage and add its downstream_tasks to the downstream vector. Now the arms for OneDone OneSuccess OneFailed.

```rust
match self.graph[node].run_condition {
	RunCondition::None => {
		stage.push(node);
		downstream.append(&mut downstream_tasks.clone());
	},
	RunCondition::OneDone => {
		for parent in self.graph.neighbors_directed(node, Direction::Incoming) {
			if task_instance_is_done(&self.graph[parent]) {
				stage.push(node);
				downstream.append(&mut downstream_tasks.clone());
				break;
			}
		}
	},
	RunCondition::OneSuccess => {
		for parent in self.graph.neighbors_directed(node, Direction::Incoming) {
			if matches!(self.graph[parent].execution_status, Some(ExecutionStatus::Success)) {
				stage.push(node);
				downstream.append(&mut downstream_tasks.clone());
				break;
			}
		}
	},
	RunCondition::OneFailed => {
		for parent in self.graph.neighbors_directed(node, Direction::Incoming) {
			if matches!(self.graph[parent].execution_status, Some(ExecutionStatus::Failed)) {
				stage.push(node);
				downstream.append(&mut downstream_tasks.clone());
				break;
			}
		}
	},
	// deducted \\
}
```

I deducted the remaining conditions, because as I’ve mentioned they are not really relevant as of yet. Basically the code for every arm is very similiar. We start of by iterating through the parents by using the neighbors_directed() functions with the Incoming direction. This means we get an Iterator of indices for the graph for all nodes with edges going towards our current node: the parents. Then we check if the parent’s execution status is meeting our run condition. If so, we add the node to the stage and its downstream_tasks to the downstream.

Simple as that, we created the current execution stage. Finally, we return the result.

```rust
if stage.len() == 0 {
	None
} else {
	Some(stage)
}
```
By returning None when the vector is empty, we signal the end of the iterator.
Review

We achieved our goal of implementing a workflow graph, which yields the tasks which are executing or up for execution. Since I’m new to Rust I’m not sure if I’ve taken the best or fanciest path, so I’m open for feedback. I’m also new to writing these kinds of posts, so feedback for this format is also appreciated.

Now that we have a Dag which we can ask for the next tasks in line, I will spend some more time on the scheduler. The next post will thus most likely be about the workings of the scheduler and using this implemention of a Dag.

Also I might come back to this implementation to add more features, refactor with more feedback and knowledge and most definetly add some logging capabilities📝.

Subscribe

    Milchdealer
    Teraku

Writing about data, programming, learning and things that are interesting enough for me to research upon and share with the world🌍.

