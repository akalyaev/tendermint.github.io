<template><div><h1>Byzantine Consensus Algorithm</h1>
<p><em>The draft 0.6 whitepaper is outdated. The new algorithm is detailed below.  See <a href=#revisions>revisions</a></em></p>
<h2>Terms</h2>
<ul>
<li>The network is composed of optionally connected <em>nodes</em>.  Nodes directly connected to a particular node are called <em>peers</em>.</li>
<li>The consensus process in deciding the next block (at some <em>height</em> <code>H</code>) is composed of one or many <em>rounds</em>.</li>
<li><code>NewHeight</code>, <code>Propose</code>, <code>Prevote</code>, <code>Precommit</code>, and <code>Commit</code> represent state machine states of a round. (aka <code>RoundStep</code> or just &#x201C;step&#x201D;).</li>
<li>A node is said to be <em>at</em> a given height, round, and step, or at <code>(H,R,S)</code>, or at <code>(H,R)</code> in short to omit the step.</li>
<li>To <em>prevote</em> or <em>precommit</em> something means to broadcast a <router-link to=/docs/internals/block-structure#vote>prevote vote</router-link> or <router-link to=/docs/internals/block-structure#precommit-vote>precommit vote</router-link> for something.</li>
<li>A vote <em>at</em> <code>(H,R)</code> is a vote signed with the bytes for <code>H</code> and <code>R</code> included in its <router-link to=/docs/internals/block-structure#vote-sign-bytes><code>sign-bytes</code></router-link>.</li>
<li><em>+2/3</em> is short for &#x201C;more than 2/3&#x201D;</li>
<li><em>1/3+</em> is short for &#x201C;1/3 or more&#x201D;</li>
<li>A set of +2/3 of prevotes for a particular block or <code>&lt;nil&gt;</code> at <code>(H,R)</code> is called a <em>proof-of-lock-change</em> or <em>PoLC</em> for short.</li>
</ul>
<h2>State Machine Overview</h2>
<p>At each height of the blockchain a round-based protocol is run to determine
the next block. Each round is composed of three <em>steps</em> (<code>Propose</code>, <code>Prevote</code>, and
<code>Precommit</code>), along with two special steps <code>Commit</code> and <code>NewHeight</code>.</p>
<p>In the optimal scenario, the order of steps is:</p>
<pre><code>NewHeight -&gt; (Propose -&gt; Prevote -&gt; Precommit)+ -&gt; Commit -&gt; NewHeight -&gt;...
</code></pre>
<p>The sequence <code>(Propose -&gt; Prevote -&gt; Precommit)</code> is called a <em>round</em>. There may be more than one round required to commit a block at a given height.  Examples for why more rounds may be required include:</p>
<ul>
<li>The designated proposer was not online.</li>
<li>The block proposed by the designated proposer was not valid.</li>
<li>The block proposed by the designated proposer did not propagate in time.</li>
<li>The block proposed was valid, but +2/3 of prevotes for the proposed block were not received in time for enough validator nodes by the time they reached the <code>Precommit</code> step.  Even though +2/3 of prevotes are necessary to progress to the next step, at least one validator may have voted <code>&lt;nil&gt;</code> or maliciously voted for something else.</li>
<li>The block proposed was valid, and +2/3 of prevotes were received for enough nodes, but +2/3 of precommits for the proposed block were not received for enough validator nodes.</li>
</ul>
<p>Some of these problems are resolved by moving onto the next round &amp; proposer.  Others are resolved by increasing certain round timeout parameters over each successive round.</p>
<h2>State Machine Diagram</h2>
<pre><code>                            +-------------------------------------+
                            v                                     |(Wait til `CommmitTime+timeoutCommit`)
                      +-----------+                         +-----+-----+
         +----------&gt; |  Propose  +--------------+          | NewHeight |
         |            +-----------+              |          +-----------+
         |                                       |                ^
         |(Else, after timeoutPrecommit)         v                |
   +-----+-----+                           +-----------+          |
   | Precommit |  &lt;------------------------+  Prevote  |          |
   +-----+-----+                           +-----------+          |
         |(When +2/3 Precommits for block found)                  |
         v                                                        |
   +--------------------------------------------------------------------+
   |  Commit                                                            |
   |                                                                    |
   |  * Set CommitTime = now;                                           |
   |  * Wait for block, then stage/save/commit block;                   |
   +--------------------------------------------------------------------+
</code></pre>
<h2>Background Gossip</h2>
<p>A node may not have a corresponding validator private key, but it nevertheless plays an active role in the consensus process by relaying relevant meta-data, proposals, blocks, and votes to its peers.  A node that has the private keys of an active validator and is engaged in signing votes is called a <em>validator-node</em>.  All nodes (not just validator-nodes) have an associated state (the current height, round, and step) and work to make progress.</p>
<p>Between two nodes there exists a <code>Connection</code>, and multiplexed on top of this connection are fairly throttled <code>Channel</code>s of information.  An epidemic gossip protocol is implemented among some of these channels to bring peers up to speed on the most recent state of consensus.  For example,</p>
<ul>
<li>Nodes gossip <code>PartSet</code> parts of the current round&#x2019;s proposer&#x2019;s proposed block.  A LibSwift inspired algorithm is used to quickly broadcast blocks across the gossip network.</li>
<li>Nodes gossip prevote/precommit votes.  A node NODE_A that is ahead of NODE_B can send NODE_B prevotes or precommits for NODE_B&#x2019;s current (or future) round to enable it to progress forward.</li>
<li>Nodes gossip prevotes for the proposed PoLC (proof-of-lock-change) round if one is proposed.</li>
<li>Nodes gossip to nodes lagging in blockchain height with block <router-link to=/docs/internals/block-structure/commit>commits</router-link> for older blocks.</li>
<li>Nodes opportunistically gossip <code>HasVote</code> messages to hint peers what votes it already has.</li>
<li>Nodes broadcast their current state to all neighboring peers. (but is not gossiped further)</li>
</ul>
<p>There&#x2019;s more, but let&#x2019;s not get ahead of ourselves here.</p>
<h2>Proposals</h2>
<p>A proposal is signed and published by the designated proposer at each round.  The proposer is chosen by a deterministic and non-choking round robin selection algorithm that selects proposers in proportion to their voting power. (see <a href=https://github.com/tendermint/tendermint/blob/develop/types/validator_set.go#L49>implementation</a>)</p>
<p>A proposal at <code>(H,R)</code> is composed of a block and an optional latest <code>PoLC-Round &lt; R</code> which is included iff the proposer knows of one.  This hints the network to allow nodes to unlock (when safe) to ensure the liveness property.</p>
<h2>State Machine Spec</h2>
<h3>Propose Step (height:H,round:R)</h3>
<p>Upon entering <code>Propose</code>:</p>
<ul>
<li>The designated proposer proposes a block at <code>(H,R)</code>.</li>
</ul>
<p>The <code>Propose</code> step ends:</p>
<ul>
<li>After <code>timeoutProposeR</code> after entering <code>Propose</code>.                --&gt; goto <code>Prevote(H,R)</code></li>
<li>After receiving proposal block and all prevotes at <code>PoLC-Round</code>. --&gt; goto <code>Prevote(H,R)</code></li>
<li>After <a href=#common-exit-conditions>common exit conditions</a></li>
</ul>
<h3>Prevote Step (height:H,round:R)</h3>
<p>Upon entering <code>Prevote</code>, each validator broadcasts its prevote vote.</p>
<ul>
<li>First, if the validator is locked on a block since <code>LastLockRound</code> but now has a PoLC for something else at round <code>PoLC-Round</code> where <code>LastLockRound &lt; PoLC-Round &lt; R</code>, then it unlocks.</li>
<li>If the validator is still locked on a block, it prevotes that.</li>
<li>Else, if the proposed block from <code>Propose(H,R)</code> is good, it prevotes that.</li>
<li>Else, if the proposal is invalid or wasn&#x2019;t received on time, it prevotes <code>&lt;nil&gt;</code>.</li>
</ul>
<p>The <code>Prevote</code> step ends:</p>
<ul>
<li>After +2/3 prevotes for a particular block or <code>&lt;nil&gt;</code>.           --&gt; goto <code>Precommit(H,R)</code></li>
<li>After <code>timeoutPrevote</code> after receiving any +2/3 prevotes.        --&gt; goto <code>Precommit(H,R)</code></li>
<li>After <a href=#common-exit-conditions>common exit conditions</a></li>
</ul>
<h3>Precommit Step (height:H,round:R)</h3>
<p>Upon entering <code>Precommit</code>, each validator broadcasts its precommit vote.</p>
<ul>
<li>If the validator has a PoLC at <code>(H,R)</code> for a particular block <code>B</code>, it (re)locks (or changes lock to) and precommits <code>B</code> and sets <code>LastLockRound = R</code>.</li>
<li>Else, if the validator has a PoLC at <code>(H,R)</code> for <code>&lt;nil&gt;</code>, it unlocks and precommits <code>&lt;nil&gt;</code>.</li>
<li>Else, it keeps the lock unchanged and precommits <code>&lt;nil&gt;</code>.</li>
</ul>
<p>A precommit for <code>&lt;nil&gt;</code> means &#x201C;I didn&#x2019;t see a PoLC for this round, but I did get +2/3 prevotes and waited a bit&#x201D;.</p>
<p>The Precommit step ends:</p>
<ul>
<li>After +2/3 precommits for <code>&lt;nil&gt;</code>.                               --&gt; goto <code>Propose(H,R+1)</code></li>
<li>After <code>timeoutPrecommit</code> after receiving any +2/3 precommits.    --&gt; goto <code>Propose(H,R+1)</code></li>
<li>After <a href=#common-exit-conditions>common exit conditions</a></li>
</ul>
<h4>common exit conditions</h4>
<ul>
<li>After +2/3 precommits for a particular block.                    --&gt; goto <code>Commit(H)</code></li>
<li>After any +2/3 prevotes received at <code>(H,R+x)</code>.                   --&gt; goto <code>Prevote(H,R+x)</code></li>
<li>After any +2/3 precommits received at <code>(H,R+x)</code>.                 --&gt; goto <code>Precommit(H,R+x)</code></li>
</ul>
<h3>Commit Step (height:H)</h3>
<ul>
<li>Set <code>CommitTime = now()</code></li>
<li>Wait until block is received.                                    --&gt; goto <code>NewHeight(H+1)</code></li>
</ul>
<h3>NewHeight Step (height:H)</h3>
<ul>
<li>Move <code>Precommits</code> to <code>LastCommit</code> and increment height.</li>
<li>Set <code>StartTime = CommitTime+timeoutCommit</code></li>
<li>Wait until <code>StartTime</code> to receive straggler commits.             --&gt; goto <code>Propose(H,0)</code></li>
</ul>
<h2>Proofs</h2>
<h3>Proof of Safety</h3>
<p>Assume that at most -1/3 of the voting power of validators is byzantine.  If a validator commits block <code>B</code> at round <code>R</code>, it&#x2019;s because it saw +2/3 of precommits at round <code>R</code>. This implies that 1/3+ of honest nodes are still locked at round <code>R&apos; &gt; R</code>.  These locked validators will remain locked until they see a PoLC at <code>R&apos; &gt; R</code>, but this won&#x2019;t happen because 1/3+ are locked and honest, so at most -2/3 are available to vote for anything other than <code>B</code>.</p>
<h3>Proof of Liveness</h3>
<p>If 1/3+ honest validators are locked on two different blocks from different rounds, a proposers&#x2019; <code>PoLC-Round</code> will eventually cause nodes locked from the earlier round to unlock.  Eventually, the designated proposer will be one that is aware of a PoLC at the later round.  Also, <code>timeoutProposalR</code> increments with round <code>R</code>, while the size of a proposal are capped, so eventually the network is able to &#x201C;fully gossip&#x201D; the whole proposal (e.g. the block &amp; PoLC).</p>
<h3>Proof of Fork Accountability</h3>
<p>Define the JSet (justification-vote-set) at height <code>H</code> of a validator <code>V1</code> to be all the votes signed by the validator at <code>H</code> along with justification PoLC prevotes for each lock change.  For example, if <code>V1</code> signed the following precommits: <code>Precommit(B1 @ round 0)</code>, <code>Precommit(&lt;nil&gt; @ round 1)</code>, <code>Precommit(B2 @ round 4)</code> (note that no precommits were signed for rounds 2 and 3, and that&#x2019;s ok), <code>Precommit(B1 @ round 0)</code> must be justified by a PoLC at round 0, and <code>Precommit(B2 @ round 4)</code> must be justified by a PoLC at round 4; but the precommit for <code>&lt;nil&gt;</code> at round 1 is not a lock-change by definition so the JSet for <code>V1</code> need not include any prevotes at round 1, 2, or 3 (unless <code>V1</code> happened to have prevoted for those rounds).</p>
<p>Further, define the JSet at height <code>H</code> of a set of validators <code>VSet</code> to be the union of the JSets for each validator in <code>VSet</code>.  For a given commit by honest validators at round <code>R</code> for block <code>B</code> we can construct a JSet to justify the commit for <code>B</code> at <code>R</code>.
We say that a JSet <em>justifies</em> a commit at <code>(H,R)</code> if all the committers (validators in the commit-set) are each justified in the JSet with no duplicitous vote signatures (by the committers).</p>
<ul>
<li><strong>Lemma</strong>: When a fork is detected by the existence of two conflicting <router-link to=/docs/internals/validators#commiting-a-block>commits</router-link>, the union of the JSets for both commits (if they can be compiled) must include double-signing by at least 1/3+ of the validator set. <strong>Proof</strong>: The commit cannot be at the same round, because that would immediately imply double-signing by 1/3+.  Take the union of the JSets of both commits.  If there is no double-signing by at least 1/3+ of the validator set in the union, then no honest validator could have precommitted any different block after the first commit.  Yet, +2/3 did.  Reductio ad absurdum.</li>
</ul>
<p>As a corollary, when there is a fork, an external process can determine the blame by requiring each validator to justify all of its round votes.  Either we will find 1/3+ who cannot justify at least one of their votes, and/or, we will find 1/3+ who had double-signed.</p>
<h3>Alternative algorithm</h3>
<p>Alternatively, we can take the JSet of a commit to be the &#x201C;full commit&#x201D;.  That is, if light clients and validators do not consider a block to be committed unless the JSet of the commit is also known, then we get the desirable property that if there ever is a fork (e.g. there are two conflicting &#x201C;full commits&#x201D;), then 1/3+ of the validators are immediately punishable for double-signing.</p>
<p>There are many ways to ensure that the gossip network efficiently share the JSet of a commit.  One solution is to add a new message type that tells peers that this node has (or does not have) a +2/3 majority for B (or <nil>) at (H,R), and a bitarray of which votes contributed towards that majority.  Peers can react by responding with appropriate votes.</nil></p>
<p>We will implement such an algorithm for the next iteration of the Tendermint consensus protocol.</p>
<p>Other potential improvements include adding more data in votes such as the last known PoLC round that caused a lock change, and the last voted round/step (or, we may require that validators not skip any votes).  This may make JSet verification/gossip logic easier to implement.</p>
<h3>Censorship Attacks</h3>
<p>Due to the definition of a block <router-link to=/docs/internals/validators#commiting-a-block>commit</router-link>, any 1/3+ coalition of validators can halt the blockchain by not broadcasting their votes.  Such a coalition can also censor particular transactions by rejecting blocks that include these transactions, though this would result in a significant proportion of block proposals to be rejected, which would slow down the rate of block commits of the blockchain, reducing its utility and value.  The malicious coalition might also broadcast votes in a trickle so as to grind blockchain block commits to a near halt, or engage in any combination of these attacks.</p>
<p>If a global active adversary were also involved, it can partition the network in such a way that it may appear that the wrong subset of validators were responsible for the slowdown.  This is not just a limitation of Tendermint, but rather a limitation of all consensus protocols whose network is potentially controlled by an active adversary.</p>
<h3>Overcoming Forks and Censorship Attacks</h3>
<p>For these types of attacks, a subset of the validators through external means
should coordinate to sign a reorg-proposal that chooses a fork (and any evidence
thereof) and the initial subset of validators with their signatures. Validators
who sign such a reorg-proposal forego its collateral on all other forks.
Clients should verify the signatures on the reorg-proposal, verify any evidence,
and make a judgement or prompt the end-user for a decision.  For example, a
phone wallet app may prompt the user with a security warning, while a
refrigerator may accept any reorg-proposal signed by +&#xBD; of the original
validators.</p>
<p>No non-synchronous Byzantine fault-tolerant algorithm can come to consensus when
&#x2153;+ of validators are dishonest, yet a fork assumes that &#x2153;+ of validators have
already been dishonest by double-signing or lock-changing without justification.
So, signing the reorg-proposal is a coordination problem that cannot be solved
by any non-synchronous protocol (i.e. automatically, and without making
assumptions about the reliability of the underlying network). It must be
provided by means external to the weakly-synchronous Tendermint consensus
algorithm.  For now, we leave the problem of reorg-proposal coordination to
human coordination via internet media.  Validators must take care to ensure that
there are no significant network partitions, to avoid situations where two
conflicting reorg-proposals are signed.</p>
<p>Assuming that the external coordination medium and protocol is robust, it follows that forks are less of a concern than <a href=#censorship-attacks>censorship attacks</a>.</p>
<h3>Revisions</h3>
<h4>0.6 -&gt; 0.7 (current)</h4>
<ol>
<li>Reduced the minimum number of signature steps from 3 to 2 by removing the &#x201C;commit&#x201D; vote and step.</li>
<li>The protocol is more asynchronous: instead of each round taking a predetermined duration of time, each step of a round progresses after +2/3 of the step&#x2019;s votes are found and a timeout is reached, or immediately after +2/3 of matching votes (e.g. a PoLC for prevotes, or a commit for precommits).</li>
</ol>
</div></template>