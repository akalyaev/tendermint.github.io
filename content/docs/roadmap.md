# Roadmap

* Allow applications (through TMSP) to update the ValidatorSet.
* Use the chain as its own CA for its nodes and validators.
* Graceful handling/recovery for apps that have non-determinism or fail to halt
* Consensus version 0.8: Update so `Commit` is not just +2/3 precommits for a block, but the entire `JSet`.  While the commit size may grow unbounded in size, it makes a fork immediately slash a +1/3 Byzantine subset of validators.
