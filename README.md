## Universal Build shell command for Command Line Tool
*if you need an iconset , you prepare an icon.png and mount to https://github.com/TokinoyuushaLink/G2-Icon-Clipper to clip your image and get an iconset

for your convenience to copy ,i place it to here:
```
#!/bin/bash
set -e

# ============================================================
# 通用 macOS App 编译脚本模板
# 使用方法：复制此文件到新项目目录，填写下方 "用户配置区" 即可
# ============================================================

# ---------- 用户配置区 ----------

APP_NAME=""
BUNDLE_ID=""
APP_VERSION="1.0.0"

# 需要 link 的 framework，按需填写，例如：
# FRAMEWORKS=(SwiftUI AppKit)
FRAMEWORKS=()

# 需要排除编译的源文件名（不参与 build 的辅助/测试脚本），例如：
# EXCLUDE_FILES=("clip_icon.swift" "Tests_logic.swift")
EXCLUDE_FILES=()

# 需要排除的目录关键字（路径中包含该字符串则跳过），例如：
# EXCLUDE_DIRS=("/文档/" "/docs/")
EXCLUDE_DIRS=()

# Info.plist 权限描述等 key-value，按需填写，没有就留空数组
# 格式："KeyName|Description文本"
# 例如：
# PLIST_EXTRA_KEYS=(
#     "NSPhotoLibraryUsageDescription|本应用需要访问您的照片以显示和整理未分类的图像。"
#     "NSCameraUsageDescription|本应用需要访问摄像头以拍摄照片。"
# )
PLIST_EXTRA_KEYS=()

LS_MIN_SYSTEM_VERSION="15.0"         # 最低系统版本
BUILD_TARGET_OS="macos15.0"          # swiftc -target 用的系统版本

# ---------- 参数 ----------

DEBUG_MODE=false
for arg in "$@"; do
    case "$arg" in
        -d|--debug) DEBUG_MODE=true ;;
    esac
done

# ---------- 配置校验 ----------

if [ -z "$APP_NAME" ] || [ -z "$BUNDLE_ID" ]; then
    echo "[错误] 请先在脚本顶部的「用户配置区」填写 APP_NAME 和 BUNDLE_ID"
    exit 1
fi

# ---------- 路径 ----------

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
APP="$SCRIPT_DIR/$APP_NAME.app"
CONTENTS="$APP/Contents"
MACOS="$CONTENTS/MacOS"
RESOURCES="$CONTENTS/Resources"
BIN="$MACOS/$APP_NAME"
ICONSET_SRC="$SCRIPT_DIR/icon.iconset"
ICON_DST="$RESOURCES/AppIcon.icns"

# ---------- 编译目标 ----------

SDK="$(xcrun --show-sdk-path)"
ARCH="$(uname -m)"
TARGET="$ARCH-apple-$BUILD_TARGET_OS"

# ---------- 开始 ----------

if [ "$DEBUG_MODE" = true ]; then
    echo "[$APP_NAME] 开始构建 (DEBUG, arch: $ARCH)"
else
    echo "[$APP_NAME] 开始构建 (arch: $ARCH)"
fi

# 1. 检查编译环境
if ! command -v swiftc &>/dev/null; then
    echo "[错误] 找不到 swiftc，请安装 Command Line Tools："
    echo "       xcode-select --install"
    exit 1
fi

# 2. 清理并重建目录结构
rm -rf "$APP"
mkdir -p "$MACOS" "$RESOURCES"

# 3. 图标生成
if [ -d "$ICONSET_SRC" ]; then
    echo "[图标] 使用 icon.iconset 生成 AppIcon.icns..."
    iconutil -c icns "$ICONSET_SRC" -o "$ICON_DST"
    echo "[图标] 完成"
else
    echo "[警告] 未找到 icon.iconset，将使用系统空白图标"
fi

# 4. 收集源文件并编译
echo "[编译] 正在收集源文件..."

# 构建 find 的排除条件
FIND_EXCLUDE_ARGS=()
for ef in "${EXCLUDE_FILES[@]}"; do
    FIND_EXCLUDE_ARGS+=(! -name "$ef")
done

SWIFT_FILES=()
while IFS= read -r f; do
    skip=false
    for ed in "${EXCLUDE_DIRS[@]}"; do
        if [[ "$f" == *"$ed"* ]]; then
            skip=true
            break
        fi
    done
    [ "$skip" = false ] && SWIFT_FILES+=("$f")
done < <(find "$SCRIPT_DIR" -name "*.swift" "${FIND_EXCLUDE_ARGS[@]}" | sort)

if [ ${#SWIFT_FILES[@]} -eq 0 ]; then
    echo "[错误] 未找到任何 .swift 源文件"
    exit 1
fi

# 构建 framework 参数
FRAMEWORK_ARGS=()
for fw in "${FRAMEWORKS[@]}"; do
    FRAMEWORK_ARGS+=(-framework "$fw")
done

echo "[编译] 共 ${#SWIFT_FILES[@]} 个文件，目标 $TARGET"
swiftc \
    -O \
    -sdk "$SDK" \
    -target "$TARGET" \
    "${FRAMEWORK_ARGS[@]}" \
    "${SWIFT_FILES[@]}" \
    -o "$BIN"

# 5. 生成 Info.plist
ICON_KEY=""
[ -f "$ICON_DST" ] && ICON_KEY="<key>CFBundleIconFile</key><string>AppIcon</string>"

EXTRA_PLIST_XML=""
for entry in "${PLIST_EXTRA_KEYS[@]}"; do
    key="${entry%%|*}"
    val="${entry#*|}"
    EXTRA_PLIST_XML="$EXTRA_PLIST_XML
  <key>$key</key><string>$val</string>"
done

cat > "$CONTENTS/Info.plist" << PLIST
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>CFBundleName</key>                <string>$APP_NAME</string>
  <key>CFBundleIdentifier</key>          <string>$BUNDLE_ID</string>
  <key>CFBundleVersion</key>             <string>$APP_VERSION</string>
  <key>CFBundleExecutable</key>          <string>$APP_NAME</string>
  <key>CFBundlePackageType</key>         <string>APPL</string>
  <key>NSPrincipalClass</key>            <string>NSApplication</string>
  <key>NSHighResolutionCapable</key>     <true/>
  <key>LSMinimumSystemVersion</key>      <string>$LS_MIN_SYSTEM_VERSION</string>
  $ICON_KEY$EXTRA_PLIST_XML
</dict>
</plist>
PLIST

# ---------- 完成 ----------

echo ""
echo "[构建成功] $APP"
echo ""
echo "按 Enter 打开应用，按 ESC 取消..."

_open_app=false
while IFS= read -rsn1 key; do
    if [[ "$key" == "" ]]; then
        _open_app=true
        break
    elif [[ "$key" == $'\x1b' ]]; then
        break
    fi
done

if [ "$_open_app" = true ]; then
    open "$APP"
fi
```
