---
layout: post
current: post
cover:  assets/built/images/background/k8s-horizontal.png
navigation: True
title: Using git submodule with AWS Code Pipeline
date: 2023-03-07 00:00:00
tags: [devops, aws]
class: post-template
subclass: 'post tag-devops'
author: HeuristicWave
---

AWS CodePipeline으로 git submodule 사용하

<br>

## Intro

여러분이 보고계신 이 블로그(GitHub Pages 활용)는 2개의 깃헙 레포지토리를 통해 배포되고 있습니다. 첫번째 레포지토리는 원본 소스코드를 담고 있으며,
블로그 글을 작성할 때 마다 `build exec jekyll exec`라는 명령어로 localhost에서 퇴고를 진행합니다.
이어서 원본 소스코드에서 `build exec jekyll build`라는 명령어로 빌드하면 빌드 결과물이 `output` 파일에 떨어집니다.
`output` 파일에는 markdown 형식으로 작성한 글들이 `html` 파일로 빌드되어 2번째 레포지토리에 커밋됩니다.



저는 이것을 자동화 하기 위해 기존에는 Travis CI를 사용하고 있었습니다.
현재 블로그로 CI/CD 파이프라인을 구축하고 약 2년간 88회의 Commit까지 잘 쓰고 있다가 어느새 다음과 같은 알람을 받아 보니, 크레딧 소진 ㅠ

> Builds have been temporarily disabled for private and public repositories due to a negative credit balance. Please go to the Plan page to replenish your credit balance.

<br>

## Outro

소중한 시간을 내어 읽어주셔서 감사합니다! 잘못된 내용은 지적해주세요! 😃

---
