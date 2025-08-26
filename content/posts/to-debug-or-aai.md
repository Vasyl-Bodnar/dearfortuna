+++
author = "V"
title = "To Debug or Apply AI"
date = "2025-08-25T23:04:15-04:00"
description = "How well do Chatbots fare against a basic bug"
tags = ["AI", "C", "Bug"]
draft = false
+++

## Intro
Recently, I have come across a peculiar issue. 
A bug most deranged. 
A kind of unspoken true evil you would not easily find in your kind and forgiving languages.

Can you find it in this cleaned-up function with totally all the necessary info?
```c
int send_data(int sd, unsigned char *txk, unsigned char *data,
              uint64_t data_len, struct header *pack,
              struct sockaddr_in *saddr) {
  struct header spack;
  const uint64_t max_size = sizeof(pack->packet) - ABYTES;
  pack->packet.off = 0;

  while (data_len > max_size) {
    memcpy(&spack, pack, sizeof(spack));
    send_pack(sd, txk, data, max_size, &spack, saddr);

    pack->packet.off += max_size;
    data_len -= max_size;
  }

  return 0;
}
```

## To Debug
### Is to find wrongdoings
Ignore the code quality for a bit.

There are only so many things that could fail in these few lines, 
but I will spare you a need to write everything around it yourself (somehow).

The issue is an infinite loop, pretty simple.

We only have one loop (ignore `send_pack` for a moment), 
so it's got to be that.
But what could be wrong? `max_size` is clearly being subtracted from `data_len`, 
and nothing else really matters for this while loop.
We could prove correctness or use a debugger, 
but it is easier to just put a few `printf`s like a true master.

Alright, after sprinkling them like ~friends~ mines on a minefield, we find a trace:
```c
data_len: 248
max_size: 512

memcpy()..
send_pack()..

data_len: 0
max_size: 512

pack->packet.off..
data_len..

data_len: 0
max_size: 512
```

### A 0 in search of a better place
Hmm, that is a `0` where it was not supposed to be. Now, this would not be surprising if we modified `data_len` somewhere. 
Maybe its address is messed with in some pointer I did not see. 
But, it is a `const`, right?

Checking the full code again.. and no. `data_len` is a `const` after all. 
There are no assignments or addresses misused.

Alright, there is clearly a change after the `memcpy` and `send_pack`. Got to be those then.
Well, the `memcpy` is fine, those are equal sizes and `sizeof` is not incorrect thankfully.

What about my `send_pack`? 
I have tested it before and it seemed to work, 
but maybe it is completely broken inside for this specific usecase.

Here it is in all its (cleanish) glory:
```c
int send_pack(int sd, unsigned char *txk, unsigned char *data,
              uint64_t data_len, struct header *pack,
              struct sockaddr_in *saddr) {
  pack->packet.size = data_len + ABYTES;
  memcpy(pack->packet.payload, data + pack->packet.off, data_len);

  hash(pack->packet.hash, sizeof(pack->packet.hash), 
       pack->packet.payload, data_len, 0, 0))

  encrypt(&pack->packet, 0, &pack->packet,
          sizeof(pack->packet) - ABYTES, 0,
          0, 0, pack->nonce, txk)

  sendto(sd, pack, sizeof(*pack), 0, (struct sockaddr *)saddr, 
         sizeof(*saddr));
  return 0;
}
```

Could look nicer, let's ignore that, not important or related at all.

Well, nothing immediately outstanding, but if you look closer.

### Const does not mean const _always_
`data_len` is used to decide the packet size and it is even in the `memcpy` and `hash`. 
Of course, in this case size refers to the payload, but what is `max_size` set to again?
```c
const uint64_t max_size = sizeof(pack->packet) - ABYTES;
```

Hmm, I guess I never showed, but `pack->packet` is the entire struct, not the payload.
```c
struct packet {
  // ...
  uint64_t off;
  unsigned char payload[200];
};

struct header {
  // ...
  struct packet packet;
};

```

So getting size of that would give you too many bytes to work with (it is larger than just the payload). 
Not so much that everything is corrupted, but a handful of things will be. 
The end result is this friend:
```c
memcpy(pack->packet.payload, data + pack->packet.off, data_len);
```
It will go too far and overwrite some stack space. 
And, of course, what comes after that `spack` we use inside of `send_pack` on the stack?
```c
struct header spack;
const uint64_t max_size =
    sizeof(pack->packet) - ABYTES;
```
So we just write too much. Stack corruption, yay!

`data` has space because `packet.off` is also incremented by `max_size`, a.k.a. `0`.
And of course we blow past `spack` into `max_size`, 
because both of them are right next to each other on stack.
`const` is just a compiler hint when you write to arbitrary stack memory.

And here I thought that using the stack was always superiour to `malloc`. Haha.

Well, this was all in the debugging build without really any optimizing, 
so what about that `-O2`. 

It just works. Ok, no it doesn't. But there is no infinite loop, 
it just sends the packets as it is supposed to and ends the program. 
Those packets are not correct and 
the other end kindly lets me know it did not like what it had to receive.

Corrupts data in some other ways (`max_size` is probably not on the stack anymore).
Thankfully fixing it (i.e. using `sizeof(pack->packet.payload)` in `max_size`) 
makes it run perfectly on either build.
At least as far as I am aware and that is good enough for prod (i.e. me).

Thus, lessons learned. ~Use a safer language~, ~write nicer code~, ask AI?

## To Apply AI
### It is not included solely for joke purposes
Indeed, it did not take me long really, 
probably not much longer than what it took you to get to here reading-wise. 
Not the hardest bug, nor the most annoying one, 
but more protein in code is rarely good.

Now I am not a vibe coder nor do I condone such violence upon the computer. 
And I would not really say I use all that much AI tooling (certainly not the latest and greatest).
Still, I find it useful to occasionally indulge in these simple chatbots that most people would use. 

Brainstorming, summarizig, finding, maybe even being of help in debugging sometimes?

So I run some basic tests, will it (GPT-5, Claude Son 4) find this bug?

### Tests for tests's sake
First the basic basic, copy and paste the entire file (~300 LOC).
Same question to both:
> Can you fix this bug for me please. `send_data` is in an infinite loop. **pasted code**

Not the finest prompt engineering, but what do I know of the arcane. 
This is realistic in that this is the info I know from just running it.
The total code size should not be an issue, it is not long, and not that dense.
Yet, both of these nice machines reply with essentially the same issue and fixes. 

Has to be those offsets. Granted they are done inside `send_pack`, 
so the bots must be confused upon seeing `send_data` without any offsets in the argument list.
They must assume that offset handling is broken and thus infinite loop somehow goes from there?

If I listened to the voices, I would be led astray.

Except what was that Claude Son 4? 
Ah right, it has randomly suggested to use `pack->packet.payload` 
in place of `pack->packet` in that cursed `sizeof`.

It was an off-hand change, but a correct solution nonetheless.
> Line 9: Also fixed max_size calculation to use sizeof(pack->packet.payload) instead of sizeof(pack->packet) - this ensures we're using the actual payload size

I did not even notice it immediately.
GPT-5 did not even compete too, it has ignored that entirely.
I had repeated trials, but given the kinds of "memory" chatbots may use, 
I am skeptical of how useful that would be.

Another test I prepared was feeding it just the essentials, 
the structs, the `send_pack`, and the `send_data`, all you need. Same question. 
(Yes, could be this mystical "memory", but it is not the exact same code at least)

GPT-5 is doing the same as it did before. 

Son 4 is still obsessed with offsets, but it has the right idea about it being `sizeof(payload)`.
Except that for some reasons it excluded `ABYTES`, authentication bytes,
those should count given that they are appended to the payload (i.e. in it)?

Ok, last test, what if I introduce it piecemeal. 

First the `send_data`, front and center. 
Same good propositions from GPT, those data offsets, underflow, 
`max_size` is 0 but that was without solutions. 
Though what do you know, GPT does not know about internals of `send_pack`.
Let us give it that next then. Same boring stuff about offsets again. 
I share the structs, and GPT finally realizes that it indeed was that `sizeof` thing.

Claude's second dearest does not perform as well as you would think given above two tests. 
Obsession over offsets given only `send_data` is expected. 
But then the same happens with `send_pack`? 
Well it probably fixed it for me according to previous function so whatever. 
Finally, the structs open its eyes and it fixes the `sizeof`, while also removing `ABYTES`. Hmm.

### To be advised by the enemy 
From these simple tests, 
you could make a quick generalization of "Claude likes big code", 
and "GPT likes chewing slowly". 
I would also like to note that GPT-5 does not produce much text at all in comparison to Son 4, 
which could be significant and even explain this difference. 
But this is no peer reviewed study across three generations.
I would recommend to not dwell on these points too much, get your own conclusions.

## And Conclusion
The result is not the most impressive, 
GPT-5 required specific massaging of pieces to get it right, which is annoying.
Also note that, if I did not know that the struct sizes were the issue, 
I would probably not give them to GPT-5. Yet they were the last piece that it needed.

Claude did better, but in weird ways. Sure it got the right answer given the full code, 
but that seems like an unnecessary divulgence (I have not hosted it on Github YET).
And when given less information, 
it seems to have gotten more lost and started removing `ABYTES` for some reason.

Both also got heavy on offsets, basically all their tokens were on that.
It probably confused them too much. 
Maybe it is wrong, but doubt is an enemy.

This is a hobby project in the end, I don't put too much focus on perfectness, 
but I do care about learning and experiencing as much as possible myself. 
Will I use AI for debugging here? 
Maybe in desperation, after hours of incorrect turns.
I have yet to get there with this project. But soon enough. 
