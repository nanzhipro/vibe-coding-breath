# Vibe Coding Breath

<!-- markdownlint-disable MD033 -->
<p align="center">
  <a href="README.md">English</a> |
  <strong>简体中文</strong>
</p>

<p align="center">
  <img src="video.gif" alt="Vibe Coding Breath 演示" width="960">
</p>
<!-- markdownlint-enable MD033 -->

**AI 工作时，你呼吸。**

`VibeCodingBreath` 是一款只驻留状态栏的 macOS 呼吸助手，专为等待 AI 完成任务的间隙而设计。
当光标静止，屏幕中央出现呼吸灯，引导吸气与呼气。
当你重新接管屏幕，它就自动隐去。

## 工作方式

- 开机自启，仅在状态栏，不占 Dock。
- 只用一枚呼吸灯，陪你吸气、呼气、回到专注。
- 把等待，变成正念练习。

## 运行方式

```bash
claude --dangerously-skip-permissions -p "$(cat PROMPT.md)"
```

## 仓库

- 规格文件：`PROMPT.md`
- 提示词仓库：[nanzhipro/vibe-coding-breath](https://github.com/nanzhipro/vibe-coding-breath)
- 生成结果仓库：[nanzhipro/VibeCodingBreath](https://github.com/nanzhipro/VibeCodingBreath)
