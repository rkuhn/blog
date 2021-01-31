# Local-First Device Management

This morning I set up a new printer at home.
For reasons that will become clear soon below, I was reminded of the first time I set up a printer in 1988:
I connected the power cord and the DB25 cable to the Epson LQ-500, hooked it up to the Atari Mega-ST2, started Signum! and very contently listened to how the bulky printer translated the parallel port’s digital pulses into tiny needle thrusts that left black marks on the paper.

33 years later the experience couldn’t be more different.
Information technology has developed astounding capabilities and changed the way we interact with our planet, and it has done so by adding many layers of indirection.
Setting up the printer today entailed connecting the power cord, going to the manufacturer’s website, downloading of the order of 100 million bytes of software, starting it, and anxiously following its completely opaque progress — abiding by the vague requirement to “stay in the vicinity of the printer”.
At some point the printer made a sound and some restarts later it appeared in the system settings.
I could now use the downloaded program to configure the printer, but I soon noticed that that only works via the cloud, demanding a working internet connection.
The printer has a built-in web server that allows local access, but getting there is most inconvenient because my web browser keeps on insisting that the authenticity of that “website” cannot be ascertained;
less determined users will simply give up.

The stark contrast between these two experiences caused me to appreciate the growth in complexity that end-users have to dig through today when things are not working.
Coming from the background of [local-first cooperation](https://local-first-cooperation.github.io/website/), the follow-up question is:
How *should* this work?
What are the absolutely required complexities and what is decorum?
In the following we’ll attempt an approximation of an answer.

## Ownership and physical access

In my household there are two kinds of electronic devices:
those with lots of customisation and therefore a strong connection to my person, and all the rest.
The first category comprises my phone, tablet, and laptop whereas the second category ranges from the central heating controller over chargers, printers, headphones, powerline adapters to set-top boxes and the internet router.
The categories only differ in the fact that I exert a level of ownership over the former that I need to explicitly relinquish for another person to make use of them, whereas the latter can just be reappropriated by anyone who lays their hands on them — they all have a factory reset procedure or an admin password printed on the back.

From this viewpoint I should be able to take my printer into operation by just having it in front of me, plugging in cables and pressing buttons.

## Administrative power

For a simple device or one that has a user interface for human interaction, it is reasonable to treat me — the one who took it into operation — as the person from whom the device shall accept commands.
This can be secured by a PIN that I choose during setup so that someone else will have to perform a factory reset before they can control the device.

In our networked world of many devices this frequently is not enough.
The printer, for example, should allow my laptop to send it print jobs, but it initially has no way of identifying the laptop and linking it to me.
Once this link is established, I will be able to print but also to perform more complex workflows like changing configuration settings or telling the printer whom else it should serve or trust.

So how shall we bootstrap this process?

First, every device needs an unforgeable identity.
As cryptography and cryptanalysis evolve, the implementation details will keep changing over the decades.
Currently, a common choice is to tie the identity to an ed25519 private key and use the corresponding public key as the “name” for the device.
This way each communication session between two devices can be authenticated by an ECDH key exchange in which each side proves to the other that it possesses the needed private key for their name without revealing that key.

Second, we shall make use of physical proximity to initiate a communication:
I might press some buttons on the printer to tell it to await an incoming session, then initiate the session from my laptop (assuming something akin to mDNS discovery).
The printer can use the signal strength, timing, and possibly a short code to validate that the session is indeed the expected one.
Once established, the session can be used to initialise the trust settings of the printer to recognise my laptop in the future.
While I’m at it, I’ll probably also authorise my phone, my watch, my direct neural interface, … I digress.

## Privilege delegation

In many cases, I’ll want to enable more things to cooperate.
The printer should also allow other family members to print and tell them about ink levels — although they may not be authorised to pass along their privileges to others.
This could be done on the level of individual devices.

If passing along privileges is fine there is an interesting alternative:
giving a name to the privilege so that it can be stored and sent around.
Like for cryptographic device names this means creating an ed25519 key pair and associating the privilege with the public key as a name.
Then anyone is allowed to print who can prove by an ECDH key exchange that they possess the corresponding private key.

With this in place I could create a QR code containing the printer’s identity and the private key that authorises printing so that guests can also print if I choose to show them the code.
The downside is that from that moment on I must assume that they retain the ability to prove possession of the right to print, so I’ll have to revoke it at intervals or whenever necessary and create a new key pair for this purpose.

## Conclusion

Based on the reasoning above it is clear that local-first cooperation with household electronics is definitely possible.
We possess the cryptographic tools to make it secure and express the same behaviour as for my old LQ-500 — with the added convenience that I can now print via wifi.

This does not mean that cloud-based business models are out of the question.
Today’s printer feels more like buying the service of having a *point of presence* for printing in my home, with ink cartridges being ordered automatically.
Any number of services could be offered that are enabled by the vast resources available in the cloud, even though I personally prefer the basic function of colouring spots on paper to work without an internet connection.

The part that I want to change is the weight of physical possession.
Even if my printer is not owned but just a rented printing service, I still want to retain the right and ability to transfer ownership of my end of this deal to someone else by just handing them the device.
There is no cloud interaction necessary in such workflows, as shown above.
Local-first cooperation is not dogmatic, it only demands that those features that can work in a purely local fashion should really do so.

## Epilogue

I left some parts of this topic for later posts, most notably the data models and communication protocols involved in the interaction between devices like my laptop and printer for the purpose of exercising ownership and administration.
And also the corner we have painted ourselves into by creating an HTML/CSS/JS monster that we can only trust based on delicate interactions between web browser standards and centralised identity proofs that simply don’t work for local services.
