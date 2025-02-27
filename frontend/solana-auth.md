I want to do a good job of explaining the take-home so please ask any questions you may have either now, when you start doing it or as you do it.

You’ll have to setup a challenge-response-signature system using `@solana/wallet-adapter-react`, `@solana/web3.js`, etc in your framework of choice (next, sveltekit, whatever you know best).

So that means user/frontend requests a nonce from backend, signs it and then sends it to backend. 
Backend verifies it and now the system know (on the backend) that this user is actually the owner of the address.

Normally you’d use this flow to return a JWT or do other things. 
That is not necessary for the take-home.

Any questions, please ask away.