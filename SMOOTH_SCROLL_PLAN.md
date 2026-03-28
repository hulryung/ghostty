# Smooth Scroll 구현 계획

## 개요

ghostty의 스크롤을 행 단위 점프에서 pixel 단위 부드러운 스크롤로 변경한다.
핵심 아이디어: precision scroll 이벤트의 sub-cell 나머지(pixel)를 shader uniform으로
전달하여 모든 셀을 해당 픽셀만큼 오프셋한다.

## 시작점

- 브랜치: `smooth-scroll`
- 기반 커밋: `b839561e5` (smooth scroll 코드 없는 upstream)

## 부호 규약

```
pending_scroll_y > 0 → 셀이 아래로 이동 → 위쪽에 gap → above extra row 필요
pending_scroll_y < 0 → 셀이 위로 이동 → 아래쪽에 gap → below extra row 필요
yoff > 0 → 위로 스크롤 (history 방향)
yoff < 0 → 아래로 스크롤 (active 방향)
scrollViewport delta: Surface에서 부호 반전 (y.delta * -1)
```

---

## Phase 1: Uniform 추가 및 Shader 오프셋 적용

**목표**: pending_scroll_y 값을 셰이더에 전달하고, 모든 셀 위치에 오프셋 적용.
스크롤 시 셀이 sub-pixel 단위로 이동하는 것을 확인.

### 1-1. Renderer State에 pending_scroll_y 추가

**파일**: `src/renderer/State.zig`
- `Mouse` struct에 `pending_scroll_y: f32 = 0` 필드 추가
- Surface의 mouse.pending_scroll_y와 별개 (Surface는 f64, renderer는 f32)

### 1-2. Uniform struct에 pending_scroll_y 추가

**파일 3개 동시 수정** (반드시 레이아웃 일치):
- `src/renderer/metal/shaders.zig` — `Uniforms` 끝에 `pending_scroll_y: f32 align(4) = 0`
- `src/renderer/opengl/shaders.zig` — 동일
- `src/renderer/shaders/shaders.metal` — struct 끝에 `float pending_scroll_y;`
- `src/renderer/shaders/glsl/common.glsl` — Globals block에 `uniform float pending_scroll_y;`

### 1-3. Surface에서 renderer state로 pending 동기화

**파일**: `src/Surface.zig`
- `scrollCallback` 함수에서, 행 스크롤 처리 후:
  ```
  self.renderer_state.mouse.pending_scroll_y = @floatCast(self.mouse.pending_scroll_y);
  ```
- 키보드 입력 시(`keyCallback`) pending을 0으로 클리어

### 1-4. Renderer에서 uniform으로 전달

**파일**: `src/renderer/generic.zig`
- Critical section에서 `state.mouse.pending_scroll_y` 읽기 (boundary check 포함)
- `self.pending_scroll_y` 필드 추가 (Generic struct)
- `rebuildCells` 이후: `self.uniforms.pending_scroll_y = self.pending_scroll_y`

### 1-5. Shader에서 오프셋 적용

**Metal** (`shaders.metal`):
- `cell_text_vertex`: `cell_pos.y += uniforms.pending_scroll_y;` (glyph 위치 계산 후)
- `cell_bg_fragment`: fragment 좌표에서 pending 빼서 grid 좌표 계산
- `image_vertex`: `image_pos.y += uniforms.pending_scroll_y;`

**GLSL** (각 파일):
- `cell_text.v.glsl`: 동일한 오프셋 적용
- `cell_bg.f.glsl`: 동일한 grid 좌표 보정
- `image.v.glsl`: 동일한 오프셋 적용

### Phase 1 검증
- `seq 1000` 후 트랙패드로 스크롤
- 셀이 sub-pixel 단위로 이동하는지 확인
- 위/아래 끝에 빈 공간(gap)이 보이는 것은 정상 (Phase 2에서 해결)

---

## Phase 2: Extra Row 처리

**목표**: 스크롤로 생기는 gap을 여분 행 데이터로 채운다.

### 2-1. Cell 버퍼 확장

**파일**: `src/renderer/cell.zig`
- `resize`에서 bg_cells: `columns * (rows + 2)` (위/아래 각 1행)
- fg_rows: `rows + 4` lists (기존 rows+2에서 +2 추가)
  - [0] cursor-before
  - [1..rows] viewport rows
  - [rows+1] below extra row
  - [rows+2] above extra row
  - [rows+3] cursor-after

### 2-2. Terminal render state에서 extra row 데이터 fetch

**파일**: `src/terminal/render.zig`
- `update` 함수에서 viewport rows 처리 후:
  - `last_viewport_pin.down(1)` → below extra row
  - `viewport_pin.up(1)` → above extra row
- `has_extra_row: [2]bool` 필드 추가 (index 0=below, 1=above)
- Extra row는 dirty 추적과 별개로 매 프레임 갱신

### 2-3. Extra row 셀 빌드

**파일**: `src/renderer/generic.zig`
- `rebuildCells`에서 `pending_scroll_y != 0`일 때:
  - below extra row (grid_pos.y = row_len) 빌드
  - above extra row (grid_pos.y = row_len + 1) 빌드
  - `grid_size.y = row_len + 1` (shader가 below row 접근 가능하도록)

### 2-4. Shader에서 extra row 처리

**Metal cell_text_vertex**:
- `grid_pos.y == grid_size.y`인 경우 → y = -1.0으로 매핑 (above row)
- BG color lookup도 above row 전용 인덱스 사용

**Metal cell_bg_fragment**:
- `grid_pos.y < 0 && pending_scroll_y > 0` → above extra row BG 읽기
- below extra row는 grid_size 확장으로 자동 접근

**GLSL**: Metal과 동일하게 적용

### Phase 2 검증
- 스크롤 시 위/아래 gap이 여분 행 내용으로 채워지는지 확인
- 배경색과 텍스트 모두 정상 표시

---

## Phase 3: 경계 처리 및 안정화

**목표**: 스크롤 경계와 edge case를 안정적으로 처리.

### 3-1. Boundary check

**파일**: `src/Surface.zig` + `src/renderer/generic.zig`
- Scrollback 최상단: `pending > 0 && top_left.up(1) == null` → pending = 0
- Active area: `pending < 0 && pinIsActive(top_left)` → pending = 0
- Mouse reporting 활성 시 → pending = 0

### 3-2. Alternate screen / mouse reporting 처리

- Alternate screen + mouse_alternate_scroll: smooth scroll 비활성
- Mouse reporting 활성: pending 클리어

### 3-3. Custom shader uniform 전달

**파일**: `src/renderer/generic.zig`
- `updateCustomShaderUniformsFromState`에서:
  ```
  uniforms.pending_scroll = .{ 0, self.pending_scroll_y };
  ```

### Phase 3 검증
- Scrollback 최상단에서 더 이상 스크롤 안 됨
- Active area에서 더 이상 아래로 스크롤 안 됨
- vim 등 TUI 앱에서 정상 동작

---

## Phase 4 (선택): 패딩 영역 개선

### 4-1. EXTEND_UP 패딩 처리
- smooth scroll 중 padding 영역에서 above extra row 색상 사용

### 4-2. EXTEND_DOWN 패딩 처리
- grid_size 확장으로 자동 처리됨 (검증만)

---

## 빌드 & 테스트 절차

매 Phase 완료 후:
```bash
cd /Users/dkkang/dev/ghostty-src
rm -rf .zig-cache macos/build

# 1. xcframework 빌드
zig build -Demit-macos-app=false -Dxcframework-target=native

# 2. ar로 모든 .a 합치기 (libtool dead-strip 우회)
WORK=$(mktemp -d)
for lib in .zig-cache/o/*/lib*.a; do
    arch=$(lipo -info "$lib" 2>/dev/null | grep -o "arm64")
    [ "$arch" = "arm64" ] || continue
    dir="$WORK/$(basename $lib .a)_$$"
    mkdir -p "$dir"
    cp "$lib" "$dir/"
    cd "$dir" && ar x "$(basename $lib)" 2>/dev/null && chmod 644 *.o 2>/dev/null
    cd /Users/dkkang/dev/ghostty-src
done
find $WORK -name "*.o" -exec ar rcs macos/GhosttyKit.xcframework/macos-arm64/libghostty-fat.a {} +
ranlib macos/GhosttyKit.xcframework/macos-arm64/libghostty-fat.a 2>/dev/null
rm -rf $WORK

# 3. xcodebuild
cd macos && rm -rf build && xcodebuild -target Ghostty -configuration Debug -arch arm64 ONLY_ACTIVE_ARCH=YES

# 4. 실행
pkill -f ghostty; sleep 1
cp -R macos/build/Debug/Ghostty.app ../zig-out/Ghostty.app
open ../zig-out/Ghostty.app
```
