# Fork Detection and processing of misbehavior with lite clients

As explained in [link to fork accountability doc] we say that a fork is a case in which an honest process (validator, full node or lite client)
observed two commits for different blocks at the same height of the blockchain. With Tendermint consensus protocol, a fork can only
happen if Tendermint Failure Model[reference to lite client doc] does not hold, i.e., we have more than 1/3 of voting power under control
of faulty validators in the relevant validator set.
In this document we focus on detections of forks observed only by lite clients, i.e., we assume that there are no forks 
on the main chain, and that a fork is used as an attack to a particular lite client.

There are several sub-problems we will cover in this document:

- a problem of detecting a fork by an honest lite client,
  
- identifying faulty validators (and not mistakenly accusing correct validators) and
  
- processing misbehavior on the main chain (if possible).  

## Detecting forks

A fork is defined as a deviation (of list of blocks) from the main chain. In this document we will assume that there exists 
a main chain C1 (sequence of blocks) and an honest full node F that is synced to the chain C1. Then a fork corresponds 
to an existence of a commit for block B for height h that is different from a block at height h
on the chain C1. Furthermore, a commit for the block B is a "valid" fork if it contains at least a single valid signature 
by a validator from the validator set at height h at chain C1. This definition of valid fork is needed to differentiate
forks that can provide material for punishment mechanisms (slashing) from bogus data that looks like a valid commit
but is signed by processes that are not tracked by the system (and therefore can't be punished).

As explained in [lite-client.md] a lite client establish a connection with a full node and it verifies application states 
of interest by downloading and verifying corresponding block headers. To be able to detect a fork, we assume an additional 
module (fork detection module) that runs in parallel with the core lite client module, and whose role is fork detection. 
The fork detection module maintains a list of peers (full nodes) and it periodically establishes a connection to some peer 
(called witness) and verify correctness of headers downloaded by main module (by cross-checking with headers downloaded
from witnesses). In case there are two different headers for the same height signed by overlapping validator sets,
then a client downloads corresponding commits and creates a proof of valid fork.

## Identifying faulty validators 

Once an honest lite client detects a valid fork it needs to submit it to an honest full node for further processing so
that faulty validators can be detected and punished. For the purpose of this document we assume that a lite client
is able to submit a proof of fork to an honest full node in reasonable time (how we can ensure that will be addressed in a
separate document). 

For the purpose of this document we assume existence of the following information for each commit: hash of the block, 
height, round, validator set and a set of signatures by the processes in the given validator set. An honest full node starts
the procedure that detects misbehaving validators from the following information: for some height h, there are two
conflicting commits C1 and C2, where C1 is a commit from the main chain. The detection procedure works like this:  

1) if there are processes from the set signers(C2) that are not part of C1.valset, they misbehaved as they are signing 
   protocol messages in heights they are not validators. In this case commit C2 is a self-contained evidence of misbehavior 
   for those processes that can be simply verified by every honest validators without additional information.  
   For processes from the set signers(C2) that are part of C1.valset we need additional checks:

2) if C1.round == C2.round, and some processes signed different precommit messages in both commits, then it is an 
   equivocation misbehavior. Similarly as above, we can easily create an evidence of misbehavior. Note that in this case
   there might be additional misbehavior, but its detection require more complex detection procedure that we explain next.  

3) if C1.round != C2.round we need to run full detection procedure. We assume (in the first version) that the full detection
   procedure is executed by a centralized, trusted component called monitor that will be processing proof of forks and 
   identifying faulty processes (and optionally generating evidences of misbehavior). Monitor is an honest full node that runs 
   on the main chain. Monitor is triggered by receiving two conflicting commits where C1.round != C2.round. The procedure starts
   by declaring processes that are in the set C1.valset as suspected. This can be for example done by posting a special transaction
   on the main chain. After suspected processes are declared, they are obliged to submit votes (prevote and precommit) for 
   the given height for the subset of rounds [C1.round, C2.round] to the monitor process within MONITOR_RESPONSE_PERIOD.
   Validators that do not submit its vote sets within this period are considered faulty (how can we create verifiable
   absence of vote sets?) 
