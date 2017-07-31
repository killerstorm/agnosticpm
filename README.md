# Agnostic Prediction Market

## Overview

The goal of AgnosticPM project is to produce a fully decentralized prediction market which works without oracles and is immune to manipulation by rich participants.

This can be done by utilizing cryptoeconomic forces similar to those which govern prices of cryptocurrencies, particularly, in case of a hard fork. We observe that a hard forked token can be used to make a bet on a particular vision of future, and reproduce this conceptual behavior within an Ethereum contract, without actual fork of a blockchain and with good UX.

Advantages of this approach:

 * no reliance of oracles, bets are settled by cryptoeconomic forces, and are thus immune to censorship, cyberattacks, etc.
 * no reliance on "democracy"
 * cannot be manipulated by wealthy entities
 * can be implemented fully within an Ethereum smart contract
 * said smart contract can be very simple and efficient
 * there is no need for a new token (no ICO!), betting tokens will be pegged to ETH (or any ERC20 token, potentially)

Disadvantages:

 * UX might be somewhat inferior to more centralized approaches
 * user might need to wait before he can withdraw his money
 * ability to withdraw depends on continuous interest in making new bets
 * better UX requires somewhat-centralized portals (which can be further refined as a separate project)
 * can only by used for betting on widely known events (obscure local events are problematic)

### History

Research of _forking decentralized prediction markets_ was started by Alex 'killerstorm' Mizrahi in 2013. The approach is described in [Decentralized Prediction Market without Arbiters](https://arxiv.org/abs/1701.08421) academic paper by Bentov, Mizrahi, Rosenfeld (published in 2017).

AgnosticPM is a specific instance of _forking DPM_ concept optimized for use in an Ethereum contract which focuses on usability.

An intermediate step between the original FDPM and AgnosticPM is [PredictionClub](https://gist.github.com/killerstorm/1d43082a03e74008a69a3a331d69d725#towards-trustless-prediction-markets--predictionclub), which uses different mechanics, but helped to find a solution for usability problems.

### Name

'Agnostic' means "without knowledge" in (Ancient?) Greek. It is a fitting name because internally the market contract doesn't know which of the outcomes have actually happened in reallity (i.e. which side have won). A contract _cannot know_ that because it exists within the blockchain and isn't in any way connected with the outside world. (Users can observe the real world, however, and can thus know the value of tokens. The contract itself doesn't need to know it.)

It's also a pun on a name of well-known Ethereum-based [Gnosis](https://gnosis.pm/) prediction market system.

## Basic mechanics

### Concepts

Following concepts apply to prediction markets in general and are not specific to AgnosticPM:

 * **Outcomes & events**. Prediction market participants can make a bet on an outcome of a certain event. For example, an event might be "Donald Trump wins 2016 US presidential election", and possible outcomes are "Yes" and "No". Alternatively, an event might be formulated like "Winner of 2016 US presidential elections", and outcomes are "Trump", "Hillary", "Other". In any case, outcomes must be mutually exclusive and jointly exhaustive, i.e. only one outcome will eventually be realized.
 * **Tokens**. We associate a distinct kind of tokens with every outcome of every event. I.e. token kind can be identified as a pair: `(event_id, outcome_id)`, for example, we can talk about `("Donald Trump wins 2016 US presidential election", "Yes")` tokens. These tokens are trade-able and (typically) transferable.
 * **Settlement**. Once an outcome of an event is known, winning tokens corresponding to it should have full value (e.g. 1 ETH per winning token), while all other losing tokens should have zero value. Prediction market should allow token owners to withdraw money they won.
 * **Token creation aka splitting**. In theory, a combination of 1 "Yes" token and 1 "No" token should always be worth 1 unit of money, because only one token will eventually win give 1 unit of money to the owner, while losing token will be worth nothing. Thus we can allow users to split 1 ETH into 1 Yes-token and 1 No-token (or, more generally, into a collection of tokens associated with event outcome).

Let's consider an example to see how a user can win a bet using a prediction market. Suppose Alice believes that Trump will win in 2016 US presidential elections, while the market "believes" that the probability of that happening is only 50%. In that case "Yes" tokens will be traded for about 0.5 ETH. Thus Alice can buy 2 Trump-Yes tokens for 1 ETH. Once the results of election are known, these tokens will be worth 1 ETH each, thus Alice effectively wins 1 ETH, doubling her money.

If market liquidity is poor, token price might not add up to 1 ETH. E.g. suppose Yes tokens price is 0.5 ETH, but No token is 0.51 ETH. In that case it will be profitable for Alice to split her 1 ETH into Yes and No tokens and sell No tokens on the market. She can then use proceedings to buy more Yes tokens. In that case she will earn 1 ETH by winning the bet + 0.1 ETH by arbitraging market inefficiency.

### Settlement mechanism

In centralized prediction markets, a market operator makes it possible to redeem a winning token for its full monetary value. In decentralized prediction markets, this is usually implemented using oracles: some sort of an oracle injects information about which token have won into a market contract, and then contract allows owners of winning tokens to withdraw their money.

Such oracles can be decentralized, e.g. there might be 5 different oracles and signatures of 4 of them will be necessary to perform a settlement. Alternatively, settlement might involve a large group of people, e.g. owners of reputation tokens (Augur). But the problem with this designs is that oracles can be compromised, or collude to steal money from winners. And voting is prone to manipulation, as one entity might buy a supermajority of voting tokens to steal from winning token holders. Also voting-based design have performance and complexity issues.

AgnosticPM does not declare tokens as winning or losing, instead, it expects traders who buy and sell tokens to know the difference: traders can observe real world events, and thus it's up to them to recognize whether tokens are winning and price them accordingly.

But why would a trader ever buy winning tokens? Even if their price _should_ be 1 ETH, if tokens have no use and cannot be redeemed for 1 ETH, they will be worthless.

In AgnosticPM, winning tokens are useful because they can be used for betting on subsequent events. I.e. 1 ETH worth of winning tokens can be used in lieu of 1 ETH to place a bet on future event. Thus, assuming that there is a demand on betting on future events, winning tokens should be worth about 1 ETH.

In theory, it's enough to track every token's betting history to make it work, as that's enough for traders to identify which tokens can be used to bet on future events and which are useless losing tokens. We assume that winning tokens will be a Schelling point of trader's buying preference. However, tracking individual token histories has certain problems:

 * it's inefficient (as there might be many tokens with slightly different histories)
 * it has bad UX, as traders will have to inspect each token's history before buying
 * it heavily relies on game-theoretic assumptions

AgnosticPM solves this problem by merging token histories where possible. This is outline in the next section.

It's worth noting that the original FDPM design also allowed mutually agreed settlement -- betting tokens could be combined back into monetary unit, e.g. 1 _Yes_ token and 1 _No_ token can be combined into 1 ETH. While, in theory, it allows better liquidity and better price tracking, possibility of this settlement makes design more complex from game-theoretic point of view, as now traders have multiple choices. Thus this option is absent from AgnosticPM design. (Although it might be introduced back later if it will be known that it's not disruptive.)

### Tracking history

As mentioned above, we need to allow traders to recognize tokens which have betting histrories incompatible with reality, but tracking it on per-token basis is wasteful. Also we want as few distinct histories as possible for efficiency and ergonomics reasons.

To solve this issue, we track history on event level. When a new bettable event is created, creator can commit to winning outcomes of previous events. E.g. when someone creates an event for betting on results of 2016 US presidential elections, he will also commit to outcome of 2012 US presidential elections, which is already known at that time.

Thus a bet is not placed on a single specific event, it is placed on a particular _timeline_ which consists of multiple events, some of which are already in the past. Of course, events which are already in the past are irrelevant for betting, but they affect traders' token choice.

Thus an event description contains a description of event proper and a list of past outcomes event commits to. These outcomes do not need to be in the same category: a presidential election event might commit to football event outcomes. Outcome commitments are transitive: if an event commits to 2016 football supercup results, it transitiviely commits to 2015, 2014 and so on results (assuming that football events always commit to the previous results).

Outcome commitments make respectable tokens compatible. E.g. suppose `Trump 2016` event commits to `(Obama 2012, Yes)` outcome. This means that users who own `(Obama 2012, Yes)` winning tokens will be able to make bets on `Trump 2016` event using their tokens instead of ether. If `Trump 2016` also commits to football outcomes, owners of winning football tokens will be able to vote on election results.

On the token level, we only keep track of the last event and outcome a bet was placed on. Tokens betting on same event outcome are considered fungible. A prior history is essentially erased: it doesn't matter if a previous bet was placed on elections or football if timelines is assumed to be compatible.

A contract can automatically verify consistency of outcome commitments and reject events which have inconsistent commitments. (E.g. if an event commits to outcome `(A, Y)` and `(B, Y)`, and `A` commits to `(C, Y)`, while `B` commits to `(C, N)` then such a commitment is transitively inconsistent.) Inconsistent commitments are bad because they violate token conservation rules. However, there are arguments against automatic enforcement on contract level:

 * this makes contract code more complex and event registration more expensive
 * commitment consistency can be trivially checked on the client side; user agent might simply ignore events and tokens with inconsistent commitments, thus the do no harm
 * there might situations in which it might be practical to ignore inconsistencies (e.g. if some event used a wrong outcome commitment due to an operator mistake or a disputed outcome, it might make a large number of tokens to be incompatible with the rest; the adverse effect of this incompatibility might be worse than little inflation)

Thus we decide to not implement automatic consistency verification on the contract level, and instead handle it on the client side.

## Practical use

### Curators

Our goal is to make prediction market which is not just theoretically sound, but is also usable. Requiring traders to manually inspect outcome committments is not ergonomic.

To simplify user experience, we introduce a concept of curators. The role of curator is verify that event meets quality standards -- has a clear description, isn't a duplicate of another event, has good structure of outcome commitments. Thus a curator will provide end users a list events they can bet on. Curators should make sure that new events commit to outcomes of past events, so users can reuse their winning tokens.

Thus curators can hide complexity of the underlying contract from the end users -- if a user selects a reputable curator, he can simply make bets on suggested events as he wishes, reusing winning tokens whenver possible. Typically users will stick to reputable, established curators, which will thus have the best token liquidity.

Curators do not exist on the prediction market contract level, at that level we only see individual events and betting tokens. Curators exist within the user interface. Any protocol can be used to retrieve a list of curator-recommended events -- it can be done via HTTP or via a separate data feed contract (which is in no way linked from the main PM contract).

Curators can register events themselves, or they can use events registered by other curators, or other parties. The contract will provide entities who create events an opportunity to charge a small fee for making a bet. This creates an incentive for curator to create high-quality events to attract as many users as possible.

#### Curators vs oracles

Aren't curators same as oracles?

There is some similarity, in a way -- curators feed information about the real world to the contract when they register an event with outcome committments.

But there is an important difference: curators do not control the settlement. Once an event is created, it doesn't depend on curator's activity.

Thus if a curator is compromised or disappears, it won't directly affect any of existing events. A user can switch to a different curator at any time.

If a curator tries to cheat by adding an wrong outcome committment to an event, no harm is done before users make a bet. Thus this problem can be mitigated by cross-checking different curators, doing manual inspection, using heuristics, etc. If user is not in a hurry, he can wait several days before placing a bet so an event is verified by a community.

Thus we conclude that curator model drastically reduces a risk and allows better mitigation strategies.

### Withdrawing money

As mentioned above, one can use his winning tokens to make subsequent bets on different events. But what if he wants to withdraw his winnings?

In principle, transferability of tokens enables a possibility of buying tokens for ether. And thus a person who wishes to make a bet might buy compatible tokens instead of splitting his ether directly.

This, however, increases the complexity and creates a question about pricing of tokens.

Instead of that, we propose propose a 'waiting queue' mechanism which simplifies UX and ensures that winning tokens are propely priced.

1. Suppose Alice owns 5 winning tokens and wishes to withdraw 5 ETH out of the contract. She adds her tokens to the withdraw waiting queue.
2. Bob wishes to split his 5 ETH to make a bet on a new event.
3. When Bob calls contract's splitting function it automatically checks for compatible tokens in the waiting queue.
4. Alice's tokens are compatible. Thus the contract simply reassigns Alice's tokens to Bob, and increases Alice's ether account by 5 ETH.
5. Now Bob has split his tokens, and Alice can withdraw 5 ETH.

Note that for Bob there is no difference whether new tokens are created to perform the splitting or tokens from the waiting queue were taken. (There might be some difference in fee, but it might actually be lower in the case of token reuse.)

As long as there is demand for making new bets, it will be possible to withdraw tokens, but the process can take some time. Users who wish to withdraw faster might sell their tokens on a secondary market with a discount, and arbitragers will eventually recover value by withdrawing ether via waiting queue mechanism.

We might also describe this mechanism as a two-way peg which pegs 1 betting token to 1 ETH, but this peg works only if there is enough demand for new bets.

## Handling funds

### Event fees

As mentioned above, we want event creators to take a small fee (say, 1%) to encourage them to create high-quality events.

It's natural to charge this fee when tokens are split. E.g. one needs 1.01 ETH to generate 1 Yes and 1 No token. 0.01 ETH will be added to event creator's account.

### Accumulated funds

Contract will accumulate funds used for bettings. These funds cannot be withdrawn by anyone since corresponding tokens always remain in the system, they cannot be used to 'back' tokens or be used as a compensation for token holders in any way -- doing so might give value to losing tokens, thus disrupting game theoretic properties of the system.

Thus the funds can be:

 * permanently locked in the contract or burned (sent to null account)
 * sent to developers as a compensation for development (this is similar to ICO model sans speculation)
 * sent to charity
 * used to compensate fees within the market

In the first version of AgnosticPM funds will be sent as a donation to developers or burned at a discretion of user. More research is needed to find better use of accumulated funds.

### Destruction bonus

What happens with losing tokens? In theory, they could just permanently remain in the system and be ignored. However, that's less than perfect, as it consumes resources of the blockchain (valuable state space) and create opportunities for token mis-use.

Instead, we can offer users a way to destroy their losing tokens and incentivize that by paying 1% bonus (in ether).

This money can come from a fee paid by users at splitting time, i.e. to split their ether or tokens into N outcomes, they need to pay `1% * (N-1)` of the sum.

(In future we might try to use accumulated funds for this.)

## Upgradeability

It is crucial for a prediction market contract to be continuously in use, as otherwise tokens will lose their value. Thus it is important to provide a contract upgrade feature which moves tokens to new contract.

At the same time, we do not want any centralized party to have control over the upgrade process, as that might be used to compromise the contract.

Thus a contract should provide a function to move user's tokens to a new contract of user's choice.

Moving accumulated funds doesn't seem to be possible.

## Further research

Following open questions and areas of improvement remain to be resolved in future versions:

 * What to do with accumulated funds? (Should we organize a large party or something?)
 * What is the best fee structre, particularly, for destruction bonus?
 * Is there a way to support token value in case of loss of interest?
 * Is there a way to improve withdrawal process or 'back' the tokens?
 * Is there a way to improve liquidity?
 * What is the best way to support upgradeability?
 * Can it work with debt-based currency? (Suggested by Dunning_Krugerrands)

However, we believe that AgnosticPM can be useful even without these improvements.

## Contract design

AgnosticPM shares a lot in common with PredictionClub design, thus we can borrow ideas from [the PredictionClub contract](https://gist.github.com/killerstorm/c0cb0e51cabd6fdc782a705cc311756d).

(History remark: PredictionClub implements event outcome committments, moving tokens from one event to another and slow withdrawal mechanics. However, it lacks forking.)

Structures:

 * OutcomeState holds users' token balances associated with a specific outcome
 * EventState holds event's outcome commitments and a list of outcomes

Fnctions which modify contract state:

 * default -- fund internal ether account
 * withdraw -- withdraw from ether account
 * registerEvent
 * splitEther -- split ether into tokens
 * upgradeAndSplit -- move tokens via committed outcomes and split
 * buyUpgradeAndSplit -- like above, but first buy tokens with ether
 * offerTokens -- start withdrawal process
 * cancelOfferTokens
 * transferTokens
 * destroyTokens




