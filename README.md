- [Interesting Fuzzing](#interesting-fuzzing)
    - [Coverage-based Greybox Fuzzing as Markov Chain(CCS 16)](#coverage-based-greybox-fuzzing-as-markov-chain(ccs-16))
    - [T-Fuzz: fuzzing by program transformation(S&P 18)](#t-fuzz:-fuzzing-by-program-transformation(s&p-18))
    - [CollAFL: Path Sensitive Fuzzing(S&P 18)](#collafl:-path-sensitive-fuzzing(s&p-18))

- [Directed Fuzzing](#directed-fuzzing)
    - [Directed Greybox Fuzzing(CCS 17)](#directed-greybox-fuzzing(ccs-17))
    - [Hawkeye: Towards a Desired Directed Grey-box Fuzzer(CCS 18)](#hawkeye:-towards-a-desired-directed-Grey-box-fuzzer(ccs-18))
## Interesting Fuzzing

### Coverage-based Greybox Fuzzing as Markov Chain(CCS 16)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/CCS16_aflfast.pdf)

- Search Strategy
- Power Schedule
- 通过改变前面两个方法来使程序更大概率地走到low-density region.

### T-Fuzz: fuzzing by program transformation(S&P 18)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/oakland18_T-Fuzz.pdf)

- Fuzzer: T-Fuzz uses an existing coverage guided fuzzer to generate inputs. T-Fuzz depends on the fuzzer to keep track of the paths taken by all the generated inputs and realtime status infomation regarding whether it is "stuck". As output, the fuzzer produces all the generated inputs. Any identified crashing inputs are recorded for further anlysis.
- Program Transformer: When the fuzzer gets "stuck", T-Fuzz invokes its Program Transformer to generate tranformed programs. Using the inputs generated by the fuzzer, the Program Transformer first traces the program under test to detect the NCC candidates and then transforms copies of the program by removing certain detected NCC candidates.
- Crash Analyzer: For crashing inputs found against the transformed programs, the Crash Analyser filters false positives using a symbolic-execution based analysis technique.

#### T-Fuzz Design

- Detecting NCCs: NCCs are those sanity checks which are present in the program logic to filter some orthogonal data, e.g., the check for a magic value in the decompressor example above. NCCs can be removed without triggering spurious bugs as they are not intended to prevent bugs. This paper uses a lightweight method to find the NCCs. Firstly, they define the concept of boundary edges: the edges connecting the nodes that were covered by the fuzzer-generated inputs and those that were not. The method that find the NCCs in this paper is over-approximation, so they find two ways to prune undesired NCC condidates.
- Program Transformation: After finding NCCs, T-Fuzz should "remove" the NCCs conditions to guide the execution to the another branch. T-Fuzz transforms programs by replacing the detected NCC candidates with negated conditional jump.
- Filtering out False Positives and Reproducing Bugs: As the removed NCC candidates might be meaningful guards in the original program(as opposed to, e.g., magic number checks), removing detected NCC edges might introduce new bugs in the transformed program. Consequently, T-Fuzz's Crash Analyzer verifies that each bug in the transformaed program is also present in the original proram, thus filtering out false positives. The Crash Analyser uses a transformation-aware combination of the preconstrained tracing technique leveraged by Driller and the Path Kneading techniques proposed by ShellSwap to collect path constraints of the original program by tracing the program path leading to a crash in the transformed program.

### CollAFL: Path Sensitive Fuzzing(S&P 18)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/oakland18_collafl.pdf)

该paper主要对AFL有两个改进:

- AFL是coverage-based greybox fuzzing，它通过对源程序进行轻量级的插桩，来跟踪每次fuzzing的input覆盖哪些路径，然后将路径hash，从而判断每个input是否到达了一个新的路径，如果到达新的路径，则说明该input较好，将该input作为seed。但由于hash可能会发生collision，可能会导致某些input到达新的路径，却没有将该input作为seed。该paper主要针对这一点，采用了一个新的算法，解决了路径hash collision问题，产生的效果也是比较显著的。
- 提供了一些策略来将seed进行排序，促使fuzzer去探索没有到达的路径。具体做法就是如果某条路径有很多没有探索到的邻居分支，则对该input进行更多的变异；如果某条路径有很多没有探索到的邻居后代，则对该input产生更多的变异。还有一个策略来帮助发现更多的漏洞：如果某条路径进行更多的内存访问，则对该input产生更多的变异。

我个人认为，该论文的主要贡献是提供了一个机制来解决路径的hash collision问题，使得coverage判断更加准确。

#### AFL Coverage Measurements

AFL使用bitmap(默认64KB)来跟踪edge coverage。没一个字节都对应特定edge的hit count。AFL通过对每个basic block进行插桩，为每个basic block都随机分配一个id，当执行每条路径时，对该路径上的每个basic block都进行如下操作:

```c
cur_location= <COMPILE_TIME_RANDOM>;
shared_mem[cur_location ^ prev_location]++;
prev_location = cur_location >> 1;
```
其中上面的prev_location右移一位主要是为了区分路径A->B和B->A。由于每个basic block的id是随机分配的，所以这种hash方法很容易产生collision，特别当程序比较大的时候，collision rate也越大。

#### CollAFL's Solution to Hash Collision

CollAFL通过三种方式来解决hash collision:

1.  ![公式1](./image/s1.png)
    通过贪心算法，为每个basic block分配x和y的值，保证每条edge计算的hash值都是不同的。
2. 如果每个basic block只有一个前继basic block，即只有一条边到达该basic block，所以只需要将该basic block的id来表示该edge即可。
3. 如果前面两种方法无法解决，则动态的时候为每条边分配不同的id。


### Driller: Argumenting Fuzzing Through Selective Symbolic Execution

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/NDSS16_driller.pdf)

### VUzzer: Application-aware Evolutionary Fuzzing

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/ndss17_vuzzer.pdf)

## Directed Fuzzing

### Directed Greybox Fuzzing(CCS 17)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/CCS17_aflgo.pdf)

```c
Input: Seed Input S

repeat
    s = CHOOSENEXT(S)
    p = ASSIGNENERGY(s)    //This paper focus
    for i from 1 to p do
        s' = MUTATE_INPUT(s)
        if t' crashes then
            add s' to Sx
        else if ISINTERESTING(s') then
            add s' to S
        end if
    end for
until timeout reached or abort-signal

Output: Crashing Inputs Sx
```

类AFL的fuzzing一般步骤如上所示，该paper主要关注于ASSIGNENERGY(s)这一操作，他们通过对不同的seed s赋予不同的energy，即如果一个seed s'产生的trace距离目标基本块targetB较近，则其energy(p)就较大，基于种子s'进行的变异操作就会变多。所以该paper主要有两个contributation: 设计一套算法计算seed s'产生的trace与targetB的距离；通过模拟退火算法来为每个seed s分配energy。

### Hawkeye: Towards a Desired Directed Grey-box Fuzzer(CCS 18)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/ccs18_hawkeye.pdf)