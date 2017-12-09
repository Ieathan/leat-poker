# leat-poker

# Provably fair poker.


The list of code I am both generating, refactoring, and pulling is growing larger and larger so I will keep track of it here.

1.) The core `leat` code which is piggy backed onto a Mubot instance.

2.) The core client `leat` code which is run on the client.

3.) The Stratum which delegates work to the clients.

4.) The poker libraries.

4.) The block chain libraries (yet to be pulled from the server core).



# Explain better

Well, I have driven myself mad for almost a decade now looking for just poker, but provably fair.

This is what I have set up so fair, albiet I have much work to do in regards to breaking what I have.

**How it works**

1.) We have the stratum delegate jobs that dont need to report back too soon. (high diff)

2.) Whenever / whichever / whoever completes the job first broadcasts it to the stratum.

3.) If there is even a single game going on the stratum queries the core server which mines a modest block.

4.) Now that we have our block we use it to generate all decks for all tables. When the server hands the clients the block if they do not immidiatly (server gains in network time offset) respond with a key which they will use to generate their hash they are disconnected. If all users respond with secrets/(lucky strings) then the server allows bets, and continues to verify that the user is hashing fairly on all subsequent bets untill the game is over.

5.) Once the game is over the salting data used to generate the block, along with any unused data cards are disclosed to the clients so that they may verify the hash against the work submitted, and the past blocks/decks.


# Show me some code

You are welcome to click the links in this repo and read and contribute, there is lots and lots I have yet to do. but the Server/Client paradigm may be a little like so:


**SERVER**
```
  /*
  * The algorithm is as follows;
  * An unkown user mines a shares, we then take
  * that share hash it against the last decks of cards 
  * and thats it.              
  *
  * The blocks hence contain the following info:
  * Relative dates
  * Ledger of who won/lost
  * Provable deck generation 
  * Verification of all previous decks 
  */
  leatServer.prototype.mineBlock = share => {
    const GENESIS = 'leat';

    /* find our previous hash */
    BlockChain.findOne().sort({
      _id: -1
    }).then(last_block => {
      /* Deal with our first block (it has no previous hash) */
      const previousHash = last_block ? last_block.hash : GENESIS;

      const options = {
        timeCost: 77,
        memoryCost: 17777,
        parallelism: 77,
        hashLength: 77
      };
      const salt = crypto.randomBytes(77);

      argond.hash(previousHash + share, salt, options).then(block_hash => {

        var block = {
          share: share,
          hash: block_hash
        };

        BlockChain.create(block);

        this.games.forEach(_=>_.emit('block found', block));

        socket.emit('block found', block)

      }
      )
    }
    )

  }
```
**CLIENT**
```
   // Deck is an immutable constantly carried over from last game
   // on all tables.
   function shuffle(Deck, block, secrets) {
      /* 
      * Our encoder.
      */
      const c = require('encode-x')();
      /*
      * Everything is an immutable constant.
      *
      ******************************************************
      *                                                    *
      *         The magic happens here, an ordinary        *
      *         HMAC-SHA512 hash created LOCALLY and       *
      *         seeded by the END USER creates the         *
      *         deck and is inserted into the next block   *
      *                                                    *
      * Note we preserve shreaded deck for future shuffles *
      ******************************************************/
      const Shuffled = sha512.hmac(
        secrets, Deck
  
      ).finalize()
      ;
      Deck.shreads = 
        Shuffled.toString('hex')
      ;

      const Debloated = Object.keys(
        c.from16to52(Deck.shreads)
         .split('').reduce(
           (a,_) =>
             a[_] = _ && a
         )
      )
      ;

      Deck.forEach(()=>
        Deck.shift()
        && Deck.pop(
          base52ToCard(
            Debloated[Deck.length]
          )
        )
      )
      ;
      
    }
}
;
```


At the end of the day merging the server/clients into one and leaving http(s) all together is a must, but bootstrapping ourselves in the web with open code is the way to go.
