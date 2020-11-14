---
title: A Comprehensive Guide to Managing GPG Keys
date: 2020-01-26 20:19:53
tags:
  - gpg
  - guide
---

## What are GPG / PGP keys

First, what is the difference between GPG and PGP. Well, it's quite simple, PGP stands for Pretty Good Privacy, which is "a family of software systems developed by Philip R. Zimmermann from which OpenPGP is based." [RFC 4880](https://tools.ietf.org/html/rfc4880.html), and GPG stands for GNU Privacy Guard. Basically, GPG is the GNU implementation of PGP (or rather, the OpenPGP standard), making it open source.

Now to answer the question, GPG keys are a set of asymetric keys which are used to encrypt / authenticate / sign things. For example, these keys can be used by Git to sign commits. If you sign commits, other developers will know for sure that you created them (assuming they trust your key).

## Motivations

Every time I have to do anything with my GPG keys, I get annoyed at myself for forgetting how to use them. The problem isn't so much the forgetting part, it's the realization that I'll be forced to dig through tons of searches to eventually figure out how to do what it is that I'm trying to do. The amount of time it takes to find some clear, simple and specific instructions to do anything with GPG keys is just way too long. And yet, I think they're awesome and that everyone should use them! This post is hopefully going to make this love/hate relationship I have for GPG more into just a love relationship, because I'll be able to conveniently find what I'm looking for right here. Hopefully, someone else benefits as well!

I should also preface this with the caveat the the guide is not yet comprehensive! I'll start with how I personally use my own GPG keys, but in the future I'll add more to this guide with helpful examples, while keeping it simple to navigate so that it remains relevant.

## Recommended Usage

- One master key
- Several subkeys
- Save master secret somewhere and remove from PC
- Only use subkeys
- Short expiration

## Normal Operation

- Day to day use of keys
- Import secret keys to change things about them
- Remove master key when done
- Touch on GPG-Agent

- `gpg --list-keys`
- `gpg --list-secret-keys`

## Creating a Key Pair

- Use ed25519
- Add a note about GitHub email and GPG key email

### Adding Subkeys

- Explain why

## Removing The Master Key

- Explain why
- `gpg --delete-secret-key {key}`

## Backup Keys

### Exporting Keys

- Cover public and private keys
- `gpg -a --export {key}` (public)
- `gpg -a --export-secret-keys {key}` (private)
- `gpg -a --export-secret-subkeys {key}` (private)

## Import Keys

- Cover public and private keys
- `gpg --import {key}`
