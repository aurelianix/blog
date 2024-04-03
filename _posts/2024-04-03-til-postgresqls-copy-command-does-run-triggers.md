---
layout: post
title: "TIL: PostgreSQL's copy command does run triggers"
date: 2024-04-03 08:14 +0200
categories:
- technology
tags:
- postgres
- TIL
---
I always assumed that the `COPY` command in PostgreSQL bypassed triggers. I didn't know that, I just assumed it. It _felt_ correct. My experience with copy is bulk importing data and that usually meant creating either temporary tables as staging area or importing into new tables. In any case, it seems I've never copied data into a tables with triggers.

From now on, I will always set `session_replication_role` to `replica` for a session that I want to bypass triggers.