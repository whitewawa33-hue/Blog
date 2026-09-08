---
title: 在Astro网站添加番茄闹钟功能
published: 2026-09-05
description: '核心需求：在网站的右侧边栏添加番茄闹钟功能，并能够设定番茄数量和达标提醒。'

tags: [Markdown, 网站更新]
category: 网站更新
draft: true
---
给网站做一个番茄闹钟功能，出现在首页右边侧边栏中，每个番茄时间为25分钟，休息5分钟，并且可以指定目标番茄数量，番茄专注 和开始还是番茄数量达标都有音效提示，但是番茄数量达标使用的音效不一样。
##  一、新建计时器组件（核心逻辑）
创建路径:`src/components/widget/Pomodoro.svelte`
```svelte
  <script lang="ts">
        /**
         * 番茄钟组件 - 首页右侧边栏
         * 专注时长默认 25 分钟（可用 +/− 按钮按 5 分钟一档手动调节），休息 5 分钟
         * 可设置目标番茄数；状态存 localStorage，刷新后自动恢复
         */
        import { onMount } from "svelte";

        // 专注/休息默认时长（分钟）
        const FOCUS_MINUTES = 25;
        const BREAK_MINUTES = 5;
        const BREAK_SECONDS = BREAK_MINUTES * 60;

        // 手动调节专注时长的档位
        const STEP = 5; // 每次加减 5 分钟
        const MIN_FOCUS = 5; // 最短 5 分钟
        const MAX_FOCUS = 120; // 最长 120 分钟

        const STORAGE_KEY = "firefly-pomodoro";

        type Mode = "idle" | "focus" | "break";

        let mode: Mode = $state("idle"); // 当前模式：idle（空闲）/ focus（专注）/ break（休息）
        // 用户自定义的专注时长（分钟），默认 25
        let focusMinutes = $state(FOCUS_MINUTES); // 自定义专注时长（分钟）
        let secondsLeft = $state(FOCUS_MINUTES * 60);
        let running = $state(false); // 是否正在倒计时
        let completed = $state(0); // 已完成番茄数
        let target = $state(4); // 默认目标数量

        let timer: ReturnType<typeof setInterval> | null = null;
        let deadline = 0; // 本轮结束时刻（时间戳，毫秒），用于跨刷新精确计时

        // ---------- 音效（Web Audio API 生成，无需音频文件） ----------
        let audioCtx: AudioContext | null = null;

        function getAudioContext(): AudioContext {
                if (!audioCtx) {
                        audioCtx = new AudioContext();
                }
                if (audioCtx.state === "suspended") {
                        audioCtx.resume();
                }
                return audioCtx;
        }

        function playTone(
                freq: number,
                startAt: number,
                duration: number,
                type: OscillatorType = "sine",
                volume = 0.3,
        ) {
                const ctx = getAudioContext();
                const osc = ctx.createOscillator();
                const gain = ctx.createGain();
                osc.type = type;
                osc.frequency.value = freq;
                const t = ctx.currentTime + startAt;
                gain.gain.setValueAtTime(0.0001, t);
                gain.gain.exponentialRampToValueAtTime(volume, t + 0.02);
                gain.gain.exponentialRampToValueAtTime(0.0001, t + duration);
                osc.connect(gain);
                gain.connect(ctx.destination);
                osc.start(t);
                osc.stop(t + duration + 0.05);
        }

        // 专注/休息开始：两声短促的“叮”
        function playStartSound() {
                playTone(880, 0, 0.15, "sine");
                playTone(1174.66, 0.15, 0.2, "sine");
        }

        // 达成目标：上升的胜利和弦（明显不同）
        function playGoalSound() {
                [523.25, 659.25, 783.99, 1046.5].forEach((f, i) => {
                        playTone(f, i * 0.15, 0.25, "triangle", 0.35);
                });
        }

        // ---------- 时长调节 ----------
        // 把数值约束在 [5, 120] 之间
        function clampFocus(v: number): number {
                if (!Number.isFinite(v)) return FOCUS_MINUTES;
                return Math.min(MAX_FOCUS, Math.max(MIN_FOCUS, v));
        }

        // 每次点击 +/− 调整 5 分钟（只在空闲状态可用）
        function adjustFocus(delta: number) {
                if (mode !== "idle") return;
                const next = clampFocus(focusMinutes + delta);
                if (next === focusMinutes) return;
                focusMinutes = next;
                secondsLeft = next * 60; // 大号倒计时同步刷新
                saveState();
        }

        // ---------- 持久化 ----------
        function saveState() {
                try {
                        localStorage.setItem(
                                STORAGE_KEY,
                                JSON.stringify({
                                        mode,
                                        running,
                                        deadline: running ? deadline : 0,
                                        remaining: running ? 0 : secondsLeft,
                                        completed,
                                        target,
                                        focusMinutes,
                                }),
                        );
                } catch {
                        // 忽略隐私模式等写入失败
                }
        }

        function loadState() {
                try {
                        const raw = localStorage.getItem(STORAGE_KEY);
                        if (!raw) return;
                        const s = JSON.parse(raw);
                        mode = s.mode === "focus" || s.mode === "break" ? s.mode : "idle";
                        completed = Math.max(0, Number(s.completed) || 0);
                        target = Math.max(1, Number(s.target) || 1);
                        const restoredMinutes = Number(s.focusMinutes);
                        focusMinutes = Number.isFinite(restoredMinutes)
                                ? clampFocus(restoredMinutes)
                                : FOCUS_MINUTES;

                        if (s.running && typeof s.deadline === "number" && s.deadline > 0) {
                                const remain = Math.ceil((s.deadline - Date.now()) / 1000);
                                if (remain > 0) {
                                        // 还没到点：按真实剩余时间继续
                                        secondsLeft = remain;
                                        deadline = s.deadline;
                                        startClock();
                                } else {
                                        // 页面关闭期间这一轮已经走完：结算一次，下一阶段从当前时刻重新开始
                                        restoreAfterElapsed();
                                }
                        } else {
                                // 空闲状态直接显示完整时长；暂停状态沿用剩余秒数
                                secondsLeft =
                                        mode === "idle"
                                                ? focusMinutes * 60
                                                : Number(s.remaining) > 0
                                                        ? Number(s.remaining)
                                                        : focusMinutes * 60;
                        }
                } catch {
                        // 读取失败则使用默认状态
                }
        }

        function restoreAfterElapsed() {
                if (mode === "focus") {
                        completed += 1;
                        if (completed >= goal) {
                                mode = "idle";
                                secondsLeft = focusMinutes * 60;
                        } else {
                                mode = "break";
                                startTimer(BREAK_SECONDS); // 休息从现在重新计时
                        }
                } else if (mode === "break") {
                        mode = "focus";
                        startTimer(focusMinutes * 60); // 用当前自定义的专注时长
                }
                saveState();
        }

        // ---------- 计时器 ----------
        function startClock() {
                if (timer) clearInterval(timer);
                timer = setInterval(tick, 250);
                running = true;
        }

        function stopClock() {
                if (timer) {
                        clearInterval(timer);
                        timer = null;
                }
                running = false;
        }

        function startTimer(durationSec: number) {
                deadline = Date.now() + durationSec * 1000;
                secondsLeft = durationSec;
                startClock();
                saveState();
        }

        function tick() {
                const remain = Math.max(0, Math.ceil((deadline - Date.now()) / 1000));
                if (remain !== secondsLeft) {
                        secondsLeft = remain;
                        saveState(); // 每秒落盘一次，刷新到哪一秒都能续上
                }
                if (remain <= 0) {
                        handleFinished();
                }
        }

        function handleFinished() {
                stopClock();
                if (mode === "focus") {
                        completed += 1;
                        if (completed >= goal) {
                                playGoalSound();
                                mode = "idle";
                                secondsLeft = focusMinutes * 60;
                        } else {
                                playStartSound(); // 休息开始
                                mode = "break";
                                startTimer(BREAK_SECONDS);
                        }
                } else if (mode === "break") {
                        playStartSound(); // 专注开始
                        mode = "focus";
                        startTimer(focusMinutes * 60); // 用当前自定义的专注时长
                }
                saveState();
        }

        // 开始专注（从空闲状态）
        function startFocus() {
                playStartSound();
                completed = 0;
                mode = "focus";
                startTimer(focusMinutes * 60);
        }

        // 暂停 / 继续
        function togglePause() {
                if (mode === "idle") return;
                if (running) {
                        secondsLeft = Math.max(0, Math.ceil((deadline - Date.now()) / 1000));
                        stopClock();
                        saveState();
                } else {
                        startTimer(secondsLeft);
                }
        }

        // 重置
        function reset() {
                stopClock();
                mode = "idle";
                secondsLeft = focusMinutes * 60;
                completed = 0;
                saveState();
        }

        const goal = $derived(Math.max(1, Math.floor(Number(target) || 1)));

        const modeLabel = $derived(
                mode === "focus" ? "专注中" : mode === "break" ? "休息中" : "空闲",
        );

        const timeClass = $derived(
                mode === "focus"
                        ? "text-(--primary)"
                        : mode === "break"
                                ? "text-emerald-500"
                                : "text-neutral-900 dark:text-neutral-100",
        );

        function format(sec: number): string {
                const m = Math.floor(sec / 60);
                const s = sec % 60;
                return `${String(m).padStart(2, "0")}:${String(s).padStart(2, "0")}`;
        }

        // 挂载时恢复状态；页面隐藏/关闭时再存一次，保证刷新不丢
        onMount(() => {
                loadState();

                const persistNow = () => saveState();
                document.addEventListener("visibilitychange", persistNow);
                window.addEventListener("pagehide", persistNow);

                return () => {
                        stopClock();
                        document.removeEventListener("visibilitychange", persistNow);
                        window.removeEventListener("pagehide", persistNow);
                };
        });
  </script>

  <div class="flex flex-col gap-3">
        <div class="text-center">
                <div class="mb-1 text-xs text-neutral-500 dark:text-neutral-400">
                        {modeLabel}
                </div>
                <div class="text-4xl font-black tabular-nums {timeClass}">
                        {format(secondsLeft)}
                </div>
        </div>

        {#if mode === "idle"}
                <!-- 专注时长调节：只在空闲时显示，点击 ±5 分钟 -->
                <div class="flex items-center justify-center gap-2 text-sm">
                        <span class="text-neutral-500 dark:text-neutral-400">专注时长</span>
                        <button
                                class="btn-plain flex h-8 w-8 items-center justify-center rounded-lg text-lg font-bold
                                        leading-none active:scale-90 disabled:pointer-events-none disabled:opacity-40"
                                onclick={() => adjustFocus(-STEP)}
                                disabled={focusMinutes <= MIN_FOCUS}
                                aria-label="减少专注时长 5 分钟"
                        >
                                −
                        </button>
                        <span
                                class="w-10 text-center font-bold tabular-nums
                                        text-neutral-900 dark:text-neutral-100"
                        >
                                {focusMinutes}
                        </span>
                        <button
                                class="btn-plain flex h-8 w-8 items-center justify-center rounded-lg text-lg font-bold
                                        leading-none active:scale-90 disabled:pointer-events-none disabled:opacity-40"
                                onclick={() => adjustFocus(STEP)}
                                disabled={focusMinutes >= MAX_FOCUS}
                                aria-label="增加专注时长 5 分钟"
                        >
                                +
                        </button>
                        <span class="text-neutral-500 dark:text-neutral-400">分钟</span>
                </div>
        {/if}

        <div class="flex items-center justify-center gap-2 text-sm">
                <span class="text-neutral-500 dark:text-neutral-400">目标</span>
                <input
                        type="number"
                        min="1"
                        max="99"
                        bind:value={target}
                        class="w-14 rounded-lg border border-neutral-300 bg-transparent px-2 py-1
                                text-center text-sm text-neutral-900 dark:border-neutral-600 dark:text-neutral-100"
                />
                <span class="text-neutral-500 dark:text-neutral-400">个番茄</span>
        </div>

        <div class="flex items-center justify-center gap-1 text-sm text-neutral-500 dark:text-neutral-400">
                <span>已完成</span>
                <span class="font-bold text-(--primary)">{completed}</span>
                <span>/ {goal}</span>
                <span>🍅</span>
        </div>

        <div class="flex items-center justify-center gap-2">
                {#if mode === "idle"}
                        <button
                                class="btn-regular rounded-lg h-9 px-4 font-bold active:scale-95"
                                onclick={startFocus}
                        >
                                开始专注
                        </button>
                {:else}
                        <button
                                class="btn-regular rounded-lg h-9 px-4 font-bold active:scale-95"
                                onclick={togglePause}
                        >
                                {running ? "暂停" : "继续"}
                        </button>
                        <button
                                class="btn-plain rounded-lg h-9 px-3 font-bold active:scale-95"
                                onclick={reset}
                        >
                                重置
                        </button>
                {/if}
        </div>
  </div>
  ```

  ##   二、新建挂件外壳（带标题的卡片）
  创建路径： `src/components/widget/Pomodoro.astro`
  ```astro
  ---
  import WidgetLayout from "@/components/common/WidgetLayout.astro";
  import Pomodoro from "@/components/widget/Pomodoro.svelte";
  import type { WidgetComponentConfig } from "@/types/config";

  interface Props {
        class?: string;
        style?: string;
        widgetConfig?: WidgetComponentConfig;
  }

  const { class: className, style, widgetConfig } = Astro.props;
  const showTitle = widgetConfig?.showTitle !== false;
  ---

  <WidgetLayout
        name="番茄钟"
        showTitle={showTitle}
        id="pomodoro"
        class={className}
        style={style}
  >
        <Pomodoro client:visible />
  </WidgetLayout>

  ---
  ```
  ## 三、在 SideBar.astro 里注册组件
  文件路径： `src/components/layout/SideBar.astro`
  ### 改动 1 —— 顶部 import 区（第 8 行 import Music ... 和第 9 行 import Profile ... 之间）：

  ```ts
  import Pomodoro from "@/components/widget/Pomodoro.astro";
  ```

  ### 改动 2 —— componentMap 对象里（第 55 行 dynamic: Dynamic, 之后）：

  ```ts   
   pomodoro: Pomodoro,
  ```
  ## 四、在右侧栏配置里开启
  文件路径： `src/config/sidebarConfig.ts`  
  ### 在 rightComponents 数组里，插到 dynamic （第 111 行）后、stats 前：
  ```ts
 {
                        // 组件类型：番茄钟组件
                        type: "pomodoro",
                        // 是否启用该组件
                        enable: true,
                        // 组件位置
                        position: "top",
                        // 是否在文章详情页显示（false = 文章页不显示，首页会显示）
                        showOnPostPage: false,
                },
  ```