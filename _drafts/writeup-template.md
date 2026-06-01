---
title: "Plataforma — Nombre de la máquina"
date: 2026-05-31 10:00:00 -0500
categories: [CTF, Linux]
tags: [ctf, linux, nmap, enumeration, privilege-escalation]
description: "Writeup técnico de la máquina X: enumeración, explotación y escalada de privilegios."
image:
  path: /assets/img/posts/nombre-maquina/preview.png
  alt: "Preview del writeup"
---

## Información general

| Campo | Valor |
|---|---|
| Plataforma | HTB / THM / Lab |
| Dificultad | Easy / Medium / Hard |
| Sistema | Linux / Windows |
| IP | 10.10.10.X |
| Técnicas | Enumeración, explotación, privilege escalation |

## Resumen

Explicación corta de qué trata la máquina y qué conceptos trabaja.

## Enumeración

```bash
nmap -sC -sV -oA scans/initial 10.10.10.X