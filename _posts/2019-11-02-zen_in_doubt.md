---
layout: post
title: The Zen is in Doubt
subtitle: Many ways to skin a cat, or, code a program.
tags: [software, python]
---

*"There should be one—and preferably only one—obvious way to do it."*
--The Zen of Python

Forgive me, for I have sinned: I doubt the Zen of Python. It's not that I think the Zen of Python is a bad set of philosophies it's just that as a junior developer I frequently suffer from design paralysis precisely because there seems to almost always be many ways to do even simple things and to me it's often non-obvious if there's a superior choice and if so, why. I suspect the issue is primarily my inexperience and not with either the language or the Zen so let's take a look at an example. 

 Let's say I want to create a class the builds a financial transaction. In my actual case this is a token for a blockchain but we can keep it simpler and more general than that. The class should allow the user to build a transaction and then call a `sign()` method to sign the transaction in preparation for it to broadcast via an API call. 

Our class will have the following parameters:
* sender
* recipient
* amount
* signer (private key for signing)
* metadata
* signed_data

For simplicity's sake we'll say all of these are strings, except for the amount which is an int, and all are required except for the last two: metadata which is an optional parameter, and `signed_data` which is c 

We would like all of the parameters to undergo some kind of validation before the signing happens so we can reject badly formatted transactions by raising an appropriate error for the user.

This seems straight-forward using a classic Python class and constructor:

```python
class Transaction:
    def __init__(self, sender, recipient, amount, signer, metadata=None):
            self.sender = sender
            self.recipient = recipient
            self.amount = amount
            self.signer = signer

            if metadata:
                self.metadata = metadata

    def is_valid(self):
        # check that all parameters are valid and exist and return True, 
        # otherwise return false

    def sign(self):
        if self.is_valid():
            # sign transaction
            self.signed_data = "pretend signature"
        else:
            # raise InvalidTransactionError
```

After signing the transaction, the Transaction object can be passed off to another function which handles broadcasting it to where it needs to go. Obviously, this is just a toy example and the real class will have more validation checks for each parameter for things like ensuring valid "sender" and "recipients", checking the amount is an integer value and within a certain range, etc. So, while this works, it has a number of disadvantages when it scales up to have more parameters. Particularly, the constructor gets long and ugly (violation of Zen!) and the `is_valid()` method will end up to quite long as well as it validates all the parameters at once. 

At this point, I'm well aware any half-decent Pythoneer reading this is pulling their hair out and screaming "properties!" and I completely agree so let's implement this class using Python's lovely `property` feature.

```python
class Transaction:
    def __init__(self, sender, recipient, amount, signer, metadata=None):
        self._sender = sender
        self._recipient = recipient
        self._amount = amount
        self._signer = signer
        self._signed_data = None

        if metadata:
            self._metadata = metadata

    @property
    def sender(self):
        return self._sender

    @sender.setter
    def sender(self, sender):
        # validate value, raise InvalidParamError if invalid
        self._sender = sender

    @property
    def recipient(self):
        return self._recipient

    @recipient.setter
    def recipient(self, recipient):
        # validate value, raise InvalidParamError if invalid
        self._recipient = recipient

    @property
    def amount(self):
        return self._amount

    @amount.setter
    def amount(self, amount):
        # validate value, raise InvalidParamError if invalid
        self._amount = amount

    @property
    def signer(self):
        return self._signer

    @signer.setter
    def signer(self, signer):
        # validate value, raise InvalidParamError if invalid
        self._signer = signer

    @property
    def metadata(self):
        return self._metadata

    @metadata.setter
    def metadata(self, metadata):
        # validate value, raise InvalidParamError if invalid
        self._metadata = metadata

    @property
    def signed_data(self):
        return self._signed_data

    @signed_data.setter
    def signed_data(self, signed_data):
        # validate value, raise InvalidParamError if invalid
        self._signed_data = signed_data

    def is_valid(self):
        return (self.sender and self.recipient and self.amount and self.signer)

    def sign(self):
        if self.is_valid():
            # sign transaction
            self.signed_data = "pretend signature"
        else:
            # raise InvalidTransactionError
            print("Invalid Transaction!")
```

This code is hella verbose and repetitive but does have some advantages. We can now validate each value when it's set so by the time we go to sign we know we have valid parameters and the `is_valid()` method only has to check that all required parameters have been set. This feels a little more Pythonic to me than doing all the validation in the single `is_valid()` method but am unsure if all the extra boiler plate code is really worth it. 

**Note: At this point it's also worth adding that it feels unpleasant to me to force the user to instantiate all the values with the constructor and I quite like giving the option of this kind of user story:**

```python
tx = Transaction()
tx.sender = "me"
tx.recipient = "you"
tx.amount = 10
tx.signer = "mykey"
```
Is this terribly un-Pythonic? I know this can be done by making all of the parameters optional by setting their default values to `None` but at this point I'm at risk of a fractal expansion of variations in this blog post so will confine this particular option to this note.**

So there are two ways to do this so far that both would work fine as far as I can tell and I only have a small preference for the latter based on fairly subjective reasons. Let's thicken the plot a little with `@dataclasses`. 

Python's `dataclasses` are the hot new thing and I've heard people gush that they should pretty much always be used now that they're on option as they get rid of a lot of boilerplate code and simplify some aspects of Python classes. Let's try to implement this as a dataclass next.

```python
@dataclass
class Transaction:
    sender: str
    recipient: str
    amount: int
    signer: str
    metadata: str = None
    signed_data: str = None

    def is_valid(self):
        # check that all parameters are valid and exist and return True, 
        # otherwise return false

    def sign(self):
        if self.is_valid():
            # sign transaction
            self.signed_data = "pretend signature"
        else:
            # raise InvalidTransactionError
            print("Invalid Transaction!")
```

Comparing this to Approach #1, this is pretty nice. It's concise, clean, and readable and already has `__init__()`, `__repr__()` and `__eq__()` methods built-in. On the other hand, compared to Approach #2 we're back to validating all the inputs via a massive `is_valid()` method. 

We could try to use properties with dataclasses but that's actually harder than it sounds. According to [this blog post][1] it can be done something like this:

```python
@dataclass
class Transaction:
    sender: str
    _sender: field(init=False, repr=False)
    recipient: str
    _recipient: field(init=False, repr=False)
   . . .
   # properties for all parameters

    def is_valid(self):
        # if all parameters exist, return True, 
        # otherwise return false

    def sign(self):
        if self.is_valid():
            # sign transaction
            self.signed_data = "pretend signature"
        else:
            # raise InvalidTransactionError
            print("Invalid Transaction!")
```

All of them seem like they will work and they all have different advantages and disadvantages. Is the Zen of Python breaking down or do am I not experienced enough to be able to make a clear choice between these approaches?

[1] https://blog.florimond.dev/reconciling-dataclasses-and-properties-in-python 