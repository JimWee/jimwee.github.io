---
title: 多场景如何正确加载遮挡剔除数据
# description: Welcome to Hugo Theme Stack
date: 2022-11-06
keywords: ["unity", "AddtiveScene", "occlusion culling data"]
image: OcclusionFrustumCulling.jpg
categories:
    - unity
tags:
    ["unity", "occlusion culling"]
---

项目中遇到一个关于同时加载多个场景时，遮挡剔除数据(occlusion culling data)没有正确加载的问题，在这个分享下解决过程和思路。

unity版本 2019.4.34f1

# 问题描述  
游戏战斗过程中会有个战斗主场景例如``BattleScene``，同时在战斗过程中可能会穿插播放几段剧情，这些剧情可能会用到其他场景资源，例如``A_CutScene``，``B_CutScene``。

实际操作过程中存在两种场景加载流程
1. 不预加载剧情场景
    - 进入战斗，使用``LoadSceneMode.Single``加载``BattleScene``
    - 播放剧情A，使用``LoadSceneMode.Additve``加载``A_CutScene``，关闭``BattleScene``里的所有物件。播放完成后卸载``A_CutScene``，开启``BattleScene``里的所有物件，回到战斗场景
    - 播放剧情B，使用``LoadSceneMode.Additve``加载``B_CutScene``，关闭``BattleScene``里的所有物件。播放完成后卸载``B_CutScene``，开启``BattleScene``里的所有物件，回到战斗场景
2. 预加载剧情场景
    - 进入战斗，使用``LoadSceneMode.Single``加载``BattleScene``，同时使用``LoadSceneMode.Additve``加载``A_CutScene``和``B_CutScene``，并关闭``A_CutScene``和``B_CutScene``里所有物件
    - 播放剧情A，使用``SetActiveScene``让``A_CutScene``变为当前激活场景，同时开启``A_CutScene``里所有物件，关闭``BattleScene``里所有物件。播放完成后卸载``A_CutScene``，开启``BattleScene``里的所有物件，回到战斗场景
    - 播放剧情A，使用``SetActiveScene``让``B_CutScene``变为当前激活场景，同时开启``B_CutScene``里所有物件，关闭``BattleScene``里所有物件。播放完成后卸载``B_CutScene``，开启``BattleScene``里的所有物件，回到战斗场景

预加载场景是为了战斗时能流畅的过度到剧情播放，不会因剧情加载卡顿，实际游戏里发现流程1里的每个场景遮挡剔除数据(occlusion culling data)是正确加载的，但流程2里遮挡剔除数据始终都是``BattleScene``的，并没有切换。

进一步实验发现，如果同时一起加载多个场景，然后使用``SetActiveScene``的方式切换当前激活场景，场景的遮挡剔除数据并不会发生切换，始终是第一个加载进来的场景的，但lightmap数据会切换🤪。

# 尝试解决
- 首先查找了[官方文档](https://docs.unity3d.com/Manual/occlusion-culling-scene-loading.html)对多场景的遮挡剔除数据加载的说明。

    大致意思是同时存在多场景要使用遮挡剔除数据的话
    1. 在bake数据阶段，先打开第一个要加载的场景（默认场景），然后打开其他addtive的场景，然后bake遮挡剔除数据，保存场景
    2. 游戏运行阶段，先加载第一个场景（默认场景），然后用``additve``的方式加载其他场景，其他场景发现遮挡剔除数据和第一个场景是一致的，就不切换遮挡剔除数据了。

    这里说明的是一个大场景由多个子场景构成的情况，然而我这边是多个场景相互切换的情况，卒。。。

- 查找文档，看有没有动态加载遮挡剔除数据的接口，没有找到。。。

- 最终只能翻看源码，找到加载遮挡剔除数据的地方，加载方式是这样的，在加载场景时（``LoadScene``的时候），会遍历当前已经加载的场景，然后加载当前激活场景的遮挡剔除数据。但``SetActiveScene``并不会触发遮挡剔除数据的重新加载。。。

# 解决方法

1. 修改源码，暴露切换遮挡剔除数据的接口，但你得有源码
2. 基于源码里切换遮挡剔除数据只发生在加载场景时，可以在``SetActiveScene``后，加载一个空场景然后立马卸载掉，就能触发遮挡剔除数据切换为当前激活的场景，虽然山寨，但不用改源码😛

各位如果有什么更好的方法，欢迎提出。
