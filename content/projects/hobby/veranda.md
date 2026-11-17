---
title: "Veranda"
build:
    render: never
params:
    url: "https://github.com/plancky/veranda"
    thumbnail: "/projects/imgs/veranda_thumbnail.png"
    tags:
        - Kotlin
        - Android
        - Jetpack Compose
        - Room
        - WorkManager
        - On-device ML
weight: 1
---

An Android app that sorts your photo library into categories (screenshots, documents, memes, people, pets, food and more) and finds near-duplicates and junk to clear out. Every photo is analyzed on the phone itself with Desert Ant Labs' Eye vision model, so nothing is ever uploaded. Scans run as a resumable background WorkManager job with results stored in Room. Each duplicate group keeps its sharpest shot, and deletes go through Android's own 30-day trash.
