---
name: gradle8-compat
description: Use this skill when the user wants to make a yutang project service compatible with Gradle 8.x, migrate a service's build files to support Gradle 8, or add Gradle version branching to a service. Triggers on phrases like "gradle8兼容", "gradle 8.x适配", "给xxx服务做gradle兼容", "升级gradle", or any request to add build_new.gradle support to a service. Also covers upgrading a service's Gradle Wrapper (gradlew/gradlew.bat/gradle-wrapper.jar/gradle-wrapper.properties) to 8.4, restoring a missing or incomplete wrapper, and un-ignoring it in .gitignore, on phrases like "升级gradlew", "wrapper升级", "gradlew换成8.4", "gradlew缺失".
version: 1.3.0
---

# Gradle 8.x 兼容适配

This skill adapts a yutang microservice to support both Gradle 4.x (via `build.gradle`) and Gradle 8.x (via `build_new.gradle`) using version-based branch switching in `settings.gradle`.

## Overview

The strategy is to keep the original `build.gradle` untouched for Gradle 4.x compatibility, and add a new `build_new.gradle` with Gradle 8.x syntax. `settings.gradle` detects the Gradle version at runtime and picks the right file. Once the build files are in place, the service's Gradle Wrapper is upgraded to 8.4 (Step 6) so the project actually runs on Gradle 8. 语法兼容做过的服务（已有 `build_new.gradle`）直接从 Step 6 开始 —— 见 Step 0。

## Step 0 — 先判断要不要做 Step 1–5

**动手前先看 `build_new.gradle` 是否已存在：**

```bash
ls build_new.gradle && grep -n 'buildFileName' settings.gradle
```

`build_new.gradle` 已存在、且 `settings.gradle` 里有版本分流块 —— 说明**语法兼容已经做过了**，Step 1–5 全部跳过，直接进 Step 6 处理 wrapper。不要重新从 `yt_pro` 拷模板覆盖，那会冲掉这个服务已经适配好的 dependencies、属性来源 URL 和 `maven_common_name`。

两者只有其一时才需要补：

| 现状 | 处理 |
|------|------|
| 两者都有 | 跳过 Step 1–5，直接做 Step 6 |
| 有 `build_new.gradle`，`settings.gradle` 缺分流块 | 只做 Step 1，然后 Step 6 |
| 有分流块，缺 `build_new.gradle` | 做 Step 2–5，然后 Step 6 |
| 两者都无 | 完整做 Step 1–6 |

本工作区多数服务的语法兼容已经做完，只差 wrapper，**Step 0 → Step 6 是最常见的路径**。

## Step 1 — Update settings.gradle

Read the service's `settings.gradle`. If the Gradle version detection block is not already present, append it after `rootProject.name = '...'`:

```groovy
def gradleVersion = GradleVersion.current().getVersion()

if (gradleVersion.startsWith("4.")) {
    println "👴 Using Gradle 4.x → Applying build.gradle"
    rootProject.buildFileName = 'build.gradle'
} else {
    println "👴👴👴 Using Gradle ${gradleVersion} → Applying build_new.gradle"
    rootProject.buildFileName = 'build_new.gradle'
}
```

If the block is already present, skip this step.

## Step 2 — 记录当前 build.gradle 的 buildscript 头部

**Do this before touching any files.** `build_new.gradle` 是从 `yt_pro` 拷来的模板，它的 buildscript 头部带着 **yt_pro 自己的**属性来源和 `maven_common_name`。这两项**不同环境/不同服务是不一样的**，必须以当前服务的 `build.gradle` 为准，不能盲用模板里的值。

从服务目录里把这两项抄下来：

```bash
sed -n '1,20p' build.gradle          # 看属性来源（String path / new File(...)）
grep -n 'maven_common_name' build.gradle
```

### 2a — 属性来源（property source）

`buildscript` 开头这段负责加载 `maven_url` / `maven_username` / `maven_password` 等变量，形态有两种：

| 形态 | 样子 | 出现在 |
|------|------|--------|
| 远程 URL | `String path = "https://download.ikuoquan.com/dev/gradle_v3_property.properties"` + `new URL(path)` 那几行 | 绝大多数服务 |
| 本地文件 | `File file = new File(project.rootDir.absolutePath + "/gradle_v3_property.properties")` + `FileInputStream` | `fission-common` 这类自带 properties 文件的 |

远程 URL 里的 **`gradle_v2_` / `gradle_v3_`** 是随环境变的关键部分，本工作区实测：

- `gg_workplace`：多数服务 `gradle_v3_property.properties`，但 `common-util`、`server-common` 是 `gradle_v2_`
- `kq_workplace` / `paojiao_workplace`：清一色 `gradle_v2_property.properties`

**所以不存在"通用值"，逐个服务看它自己的 `build.gradle`。** 不要因为上一个服务是 v3 就默认这个也是 v3。

### 2b — maven_common_name

同样以 `build.gradle` 里的实际值为准：

```groovy
String maven_common_name = "gg"
```

`build.gradle` 里已经写了就直接用它。只有在原文件里找不到时，才按分支名推断：

| Branch prefix | maven_common_name value |
|---------------|------------------------|
| `gg_*`        | `"gg"`                 |
| `kuoquan_*`   | `"kq"`                 |
| anything else | `"yt"`                 |

```bash
git rev-parse --abbrev-ref HEAD
```

注意分支推断只是兜底，会推错：`fission-common` 在 `gg_dev` 分支上，实际值却是 `"common"`。**读到的值优先于推出来的值。**

把 2a、2b 两项记下来 —— Step 4 要用。

## Step 3 — Copy build_new.gradle from yt_pro branch

The `build_new.gradle` template lives in the remote `yt_pro` branch of the same repository. Copy it:

```bash
git show origin/yt_pro:build_new.gradle > build_new.gradle
```

Run this command from inside the service's directory. If `origin/yt_pro` doesn't have this file, check for it in `origin/yt_pro` of another service in the same workspace directory (e.g., `mangement`).

## Step 4 — 用 build.gradle 的头部覆盖模板头部

模板的 buildscript 头部整段都属于 `yt_pro`，必须换成 Step 2 记录的值。**两项都要改，即使看起来已经对了** —— 模板永远来自 `yt_pro`，`gradle_v2_/v3_` 和 `"yt"` 都是它自己的。

### 4a — 属性来源

把 `build_new.gradle` 里从 `buildscript {` 之后到 `for (Map.Entry...)` 循环之前的那段加载代码，替换成 `build.gradle` 里对应的那段（原样照抄，包括注释行）。

- 两边都是远程 URL 形态时，改一行 `String path = "..."` 就够了
- 形态不同（一边 URL、一边 `new File`）时，整段换掉，别只改 URL

改完自查：

```bash
diff <(sed -n '/^buildscript/,/^    project.ext/p' build.gradle) \
     <(sed -n '/^buildscript/,/^    project.ext/p' build_new.gradle)
```

**无差异才算对齐。** 有差异就说明还留着 `yt_pro` 的值。

### 4b — maven_common_name

Find the line in `build_new.gradle`:

```groovy
String maven_common_name = "yt"
```

改成 Step 2b 记录的值（例如 `"gg"`、`"common"`）。

## Step 5 — Replace dependencies section in build_new.gradle

The `build_new.gradle` copied from `yt_pro` has its own project-specific `dependencies { }` block (lines ~73-131 in the template). This must be replaced with the current service's dependencies from `build.gradle`.

### 5a — Extract dependencies from build.gradle

Read `build.gradle` and find the `dependencies { ... }` block. Copy its entire content.

### 5b — Delete the old dependencies block from build_new.gradle

Delete from the line `dependencies {` through the matching closing `}` in `build_new.gradle`.

### 5c — Convert syntax to Gradle 8.x and insert

Apply these replacements to every line in the extracted dependencies:

| Old (Gradle 4.x) | New (Gradle 8.x)   |
|-------------------|--------------------|
| `compile `        | `implementation `  |
| `testCompile `    | `testImplementation ` |
| `runtime `        | `runtimeOnly `     |
| `compileOnly `    | `compileOnly `     (no change) |
| `annotationProcessor ` | `annotationProcessor ` (no change) |

Then insert the converted `dependencies { ... }` block into `build_new.gradle` at the position where the old block was deleted (before the `java { ... }` block).

## Step 6 — 升级 Gradle Wrapper 到 8.4

**必须在 Step 1–5 之后做。** `wrapper` 本身也是一个 task，执行前 Gradle 要先加载 `settings.gradle` 和构建脚本。下面 6a 里由 Gradle 8.x 执行的那一遍，如果版本分流和 `build_new.gradle` 没到位，会在配置阶段直接失败，根本走不到生成文件那一步。

### 6a — 重新生成四个文件

Wrapper 是**四个文件一套**，缺一不可：

```
gradlew                                   启动脚本（sh）
gradlew.bat                               启动脚本（cmd）
gradle/wrapper/gradle-wrapper.jar         引导程序，负责下载 Gradle 本体
gradle/wrapper/gradle-wrapper.properties  distributionUrl，决定用哪个版本
```

**先查现状，再决定走哪条路线：**

```bash
ls gradlew gradlew.bat gradle/wrapper/gradle-wrapper.jar gradle/wrapper/gradle-wrapper.properties
```

#### 为什么命令要跑两遍

不管走哪条路线，`wrapper` 命令都要执行两遍。因为它的产物有两个来源：

- `distributionUrl` ← 来自 `--gradle-version` 参数
- `gradlew` / `gradlew.bat` / `gradle-wrapper.jar` ← 来自**正在执行这个 task 的那个 Gradle 自带的模板**

第一遍只能保证 properties 指向 8.4，脚本和 jar 属于"执行者"而不是目标版本，停在这一步就是半吊子。第二遍 `./gradlew` 已按新 properties 把 8.4 下载并启动，执行者本身就是 8.4，四个文件才全部收敛到目标版本。（实测 8.14.4 与 8.4 生成的 `gradlew` 和 jar 并不相同，所以本机 gradle 版本再新也替代不了第二遍。）

#### 路线 A — 四个文件齐全

仓库里已有可用的 wrapper（哪怕是 3.5/4.x 的老版本），用它自己引导即可，**不需要本机安装 gradle**：

```bash
./gradlew wrapper --gradle-version 8.4    # 第一遍：由仓库现有的老 Gradle 执行
./gradlew wrapper --gradle-version 8.4    # 第二遍：此时已是 8.4 在执行
```

#### 路线 B — 有文件缺失，用本机 gradle 引导

只要 `gradle-wrapper.jar` 或 `gradle-wrapper.properties` 缺失（最常见的成因：`.gitignore` 里有 `/gradle/`，wrapper 从未入库，clone 出来就只剩两个脚本），`./gradlew` 无法引导，会直接报：

```
Error: Could not find or load main class org.gradle.wrapper.GradleWrapperMain
```

这种情况**只能先用本机安装的 gradle 把四个文件补出来**，再照常收敛：

```bash
gradle --version                          # 确认本机装了 gradle
gradle wrapper --gradle-version 8.4       # 第一遍：本机 gradle 补齐四个文件
./gradlew wrapper --gradle-version 8.4    # 第二遍：收敛到 8.4
```

本机 gradle 是什么版本都行，它只在第一遍充当引导，最终产物由第二遍的 8.4 决定。但版本决定第一遍走哪个构建脚本：本机是 8.x 就要求 `build_new.gradle` 已就绪（Step 1–5 已完成），本机是 4.x 则走原来的 `build.gradle`。

#### 路线 C — wrapper 残缺且本机没装 gradle

从已升级好的项目直接拷贝，结果完全等价 —— 这四个文件不含任何项目相关内容，唯一的差异就是 `distributionUrl` 那一行：

```bash
cp ../fission-common/gradlew ../fission-common/gradlew.bat ./
mkdir -p gradle/wrapper
cp ../fission-common/gradle/wrapper/gradle-wrapper.jar gradle/wrapper/
cp ../fission-common/gradle/wrapper/gradle-wrapper.properties gradle/wrapper/
chmod +x gradlew
```

拷完可再跑一遍 `./gradlew wrapper --gradle-version 8.4` 自检（此时执行者已是 8.4，相当于第二遍，结果应无变化）。项目配置不起来或无网络时，这条路线同样是 A/B 的兜底。

最后确认 properties 内容：

```properties
distributionUrl=https\://services.gradle.org/distributions/gradle-8.4-bin.zip
```

### 6b — gradle 目录只保留必要文件

`gradle/` 下**只应该有这两个文件**，其余一律删掉（本地缓存 `dists/`、`*.bak`/`*.orig` 备份、误放进去的 `gradle.properties` 等）：

```bash
find gradle -mindepth 1 -not -path gradle/wrapper \
    -not -name gradle-wrapper.jar -not -name gradle-wrapper.properties
```

先看清单，确认无误再删。

### 6c — 解除 .gitignore 对 wrapper 的忽略

很多服务的 `.gitignore` 把 wrapper 一起忽略了，导致仓库里的 wrapper 是**残缺的** —— 别人 clone 下来或 CI 拉完代码，`./gradlew` 会因为找不到 `gradle-wrapper.jar` 直接失败（也就是路线 B 那个报错的来源）。检查：

```bash
git check-ignore -v gradlew gradlew.bat \
    gradle/wrapper/gradle-wrapper.jar gradle/wrapper/gradle-wrapper.properties
```

**无输出才是正确的**，有输出就说明被哪条规则忽略了，按类型处理：

| 命中的规则 | 处理方式 |
|-----------|---------|
| `/gradle/` 或 `gradle/` | 直接删掉这一行 |
| `*.jar` | 保留原规则，另加一行反忽略：`!gradle/wrapper/gradle-wrapper.jar` |

**注意 `.gradle`（带点，本地构建缓存目录）那条要保留**，它和 `gradle/` 是两回事，别误删。

改完重新跑一遍 `git check-ignore`，确认无输出后加入版本库：

```bash
git add gradlew gradlew.bat gradle/wrapper/gradle-wrapper.jar \
        gradle/wrapper/gradle-wrapper.properties .gitignore
```

### 6d — 验证

```bash
./gradlew --version    # 应显示 Gradle 8.4
```

`--version` 只能证明 wrapper 可以把 8.4 拉起来，**不代表构建能过**。条件允许时再跑一次真实构建（如 `./gradlew build -x test`）确认 `build_new.gradle` 在 8.4 下可用。

## Verification

只做了 Step 6 的（Step 0 判定跳过 Step 1–5），只需确认第 5–8 条。完整跑完的确认全部：
1. `settings.gradle` contains the version detection block
2. `build_new.gradle` exists；buildscript 头部的属性来源（`gradle_v2_`/`gradle_v3_`，或本地 `new File` 形态）与本服务 `build.gradle` 完全一致，`maven_common_name` 也与 `build.gradle` 一致（不是模板里的 `"yt"`）
3. The `dependencies` block in `build_new.gradle` uses `implementation`/`testImplementation` (no `compile`/`testCompile`)
4. The rest of `build_new.gradle` (buildscript, plugins, repositories, java block, tasks) is intact
5. Wrapper 四个文件齐全，`gradle-wrapper.properties` 指向 `gradle-8.4-bin.zip`
6. `gradle/` 下除 `wrapper/gradle-wrapper.jar` 和 `wrapper/gradle-wrapper.properties` 外无其他文件
7. `git check-ignore -v` 对这四个文件无输出（即都能提交进版本库）
8. `./gradlew --version` 输出 Gradle 8.4

Report what was changed to the user.

## Notes

- Always work inside the target service's directory (not the workspace root)
- Do NOT modify `build.gradle` — it must stay as-is for Gradle 4.x
- Commented-out lines (starting with `//`) can be left as-is or also converted for consistency
- `build_new.gradle` 的 buildscript 头部（属性 URL、`maven_common_name`）永远以本服务 `build.gradle` 为准，`yt_pro` 模板里的值只是占位
- 服务已有 `build_new.gradle` 就视为语法兼容已完成，按 Step 0 跳到 Step 6；不要覆盖它，除非用户明确要求重做
- Wrapper 的四个文件全部要提交进 git，少一个别人就跑不起来；`gradle-wrapper.jar` 虽是二进制也必须入库
- 只有两个脚本、没有 `gradle/` 目录，是本工作区最常见的残缺形态，走 Step 6a 路线 B 或 C
- 如果服务原本的 `distributionUrl` 是 9.x，同样降到 8.4 —— Gradle 9 要求 Java 17+，且删除了 8.x 中已废弃的一批 API
- 只改 wrapper 不等于项目能在 8.4 下构建；真正的适配工作是 Step 1–5
