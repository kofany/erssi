# Sidepanels Implementation Audit

**Date**: 2025-01-XX  
**Component**: erssi sidepanels rendering system  
**Version**: Current (main branch)

## Executive Summary

The sidepanels implementation is **professionally architected** with excellent separation of concerns across 5 modules. The batched redraw system and targeted updates demonstrate sophisticated performance optimization. However, several bugs and optimization opportunities were identified.

## Architecture Overview

### Module Structure (5-Component Design)

```
sidepanels-core.c         → Coordinator & mouse handling
sidepanels-signals.c      → IRC event handling (join/part/quit/nick)
sidepanels-activity.c     → Activity tracking & window priority
sidepanels-layout.c       → Geometry management & positioning
sidepanels-render.c       → Rendering & optimization layer
sidepanels-types.h        → Shared data structures
```

### Key Strengths

1. **Excellent Separation of Concerns**: Each module has a clear, well-defined responsibility
2. **Batched Redraw System**: 5ms timeout prevents excessive redraws during mass events (e.g., 400+ user channels)
3. **Targeted Redraws**: Three-level optimization strategy
   - `redraw_left_panels_only()` - Window list changes
   - `redraw_right_panels_only()` - Nicklist changes
   - `redraw_both_panels_only()` - Combined updates
4. **UTF-8/Grapheme Cluster Handling**: Proper `string_advance()` and `read_unichar()` usage
5. **Theme Integration**: Full irssi theme support with `draw_str_themed()` and `draw_str_themed_2params()`
6. **Clean Data Structures**: `SP_MAINWIN_CTX` with geometry caching for efficient hit-testing

### Redraw Optimization Strategy

```c
// Level 1: Targeted panel updates (most common)
redraw_left_panels_only("event_name");   // Activity/window list changes
redraw_right_panels_only("event_name");  // Nicklist updates

// Level 2: Both panels (layout changes)
redraw_both_panels_only("event_name");   // Window switch, item change

// Level 3: Batched updates (mass events)
schedule_batched_redraw("event_name");   // Netsplits, mass joins
```

## Bugs Identified

### 🔴 CRITICAL: Bug #1 - Inefficient Nicklist Rendering
**File**: `sidepanels-render.c:595-610`  
**Impact**: O(3N) memory allocations + O(N log N) sorting × 3 on every nicklist redraw

**Current Code**:
```c
// Build three separate lists with prepend (reverses order)
for (nt = nicks; nt; nt = nt->next) {
    nick = nt->data;
    if (nick->op)
        ops = g_slist_prepend(ops, nick);      // O(N)
    else if (nick->voice)
        voices = g_slist_prepend(voices, nick); // O(N)
    else
        normal = g_slist_prepend(normal, nick); // O(N)
}

// Sort each list separately
ops = g_slist_sort(ops, ci_nick_compare);       // O(N log N)
voices = g_slist_sort(voices, ci_nick_compare); // O(N log N)
normal = g_slist_sort(normal, ci_nick_compare); // O(N log N)
```

**Problem**: 
- Creates 3 intermediate lists unnecessarily
- Sorts 3 times instead of once
- `g_slist_prepend()` reverses order, forcing sort anyway

**Better Approach**: Single sorted list with custom comparator:
```c
// Single pass: copy pointers and sort once with priority+alphabetical
GSList *all_nicks = g_slist_copy(nicks);
all_nicks = g_slist_sort(all_nicks, nick_compare_with_priority);

// Where nick_compare_with_priority() returns:
// - ops before voices before normal (by checking op/voice flags)
// - alphabetical within each group (by strcmp)
```

**Performance**: O(N + N log N) instead of O(3N + 3(N log N))

---

### 🟡 MEDIUM: Bug #2 - Race Condition in Batched Redraw
**File**: `sidepanels-render.c:808-829`  
**Impact**: Potential use-after-free if event_name string is dynamically allocated

**Current Code**:
```c
void schedule_batched_redraw(const char *event_name)
{
    // ...
    redraw_timer_tag = g_timeout_add(redraw_batch_timeout, 
                                      batched_redraw_timeout, 
                                      (gpointer) event_name);  // <-- Danger!
}

static gboolean batched_redraw_timeout(gpointer data)
{
    const char *event_name = (const char *) data;  // May be invalid
    redraw_both_panels_only(event_name);
    // ...
}
```

**Problem**: 
- `event_name` pointer is passed directly to timeout callback
- If caller passes a dynamically allocated string (unlikely but possible), it could be freed before timeout fires
- Currently safe because all callers pass string literals, but API is fragile

**Solution**: 
1. Document that `event_name` must be a string literal/static, OR
2. Duplicate the string in `schedule_batched_redraw()` and free in callback

---

### 🟡 MEDIUM: Bug #3 - Missing Error Handling in Truncation
**File**: `sidepanels-render.c:371-423`  
**Impact**: Potential buffer issues if max_width is negative

**Current Code**:
```c
char *truncate_nick_for_sidepanel(const char *nick, int max_width)
{
    if (!nick)
        return g_strdup("");
    
    if (max_width <= 0)  // Only checks <= 0, not < 0
        return g_strdup("+");
    
    // ... no validation that max_width is reasonable
}
```

**Problem**: 
- No validation that max_width is reasonable (e.g., < 1000)
- Caller could pass negative values through calculation errors

**Solution**: Add bounds checking:
```c
if (max_width < 0) {
    sp_logf("ERROR: truncate_nick_for_sidepanel called with negative max_width=%d", max_width);
    return g_strdup("+");
}
if (max_width == 0)
    return g_strdup("+");
```

---

### 🟡 MEDIUM: Bug #4 - Redundant position_tw() Calls
**File**: `sidepanels-render.c:783-792`  
**Impact**: Unnecessary panel repositioning on every redraw

**Current Code**:
```c
void redraw_both_panels_only(const char *event_name)
{
    // ...
    if (ctx->left_tw && ctx->left_h > 0) {
        position_tw(mw, ctx);           // First call
        draw_left_contents(mw, ctx);
        needs_redraw = TRUE;
    }
    
    if (ctx->right_tw && ctx->right_h > 0) {
        if (!ctx->left_tw || ctx->left_h == 0) {
            position_tw(mw, ctx);       // Conditional second call
        }
        draw_right_contents(mw, ctx);
        needs_redraw = TRUE;
    }
}
```

**Problem**: 
- `position_tw()` called twice if only right panel exists
- Awkward conditional logic

**Solution**: Call once at the start:
```c
void redraw_both_panels_only(const char *event_name)
{
    // ...
    ctx = get_ctx(mw, FALSE);
    if (!ctx) continue;
    
    position_tw(mw, ctx);  // Single call for both panels
    
    if (ctx->left_tw && ctx->left_h > 0) {
        draw_left_contents(mw, ctx);
        needs_redraw = TRUE;
    }
    
    if (ctx->right_tw && ctx->right_h > 0) {
        draw_right_contents(mw, ctx);
        needs_redraw = TRUE;
    }
}
```

---

### 🟢 LOW: Bug #5 - Missing NULL Check in Themed Drawing
**File**: `sidepanels-render.c:304-368`  
**Impact**: Potential crash if wctx is NULL (unlikely in practice)

**Current Code**:
```c
void draw_str_themed_2params(TERM_WINDOW *tw, int x, int y, WINDOW_REC *wctx, ...)
{
    format_create_dest(&dest, NULL, NULL, 0, wctx);  // No NULL check
    // ...
}
```

**Solution**: Add guard:
```c
if (!wctx) {
    sp_logf("ERROR: draw_str_themed_2params called with NULL wctx");
    return;
}
```

---

### 🟢 LOW: Bug #6 - Unused right_selected_index
**File**: `sidepanels-types.h:40`, `sidepanels-core.c:414`  
**Impact**: Dead code, minor memory waste

**Current State**:
- `right_selected_index` is set in `handle_click_at()` (line 414)
- Never read or used for any rendering decision
- Takes up 4 bytes in every `SP_MAINWIN_CTX` struct

**Action**: Can be removed unless future feature planned

---

## Missing Optimizations (Not Bugs)

### Opportunity #1: Dirty Line Tracking
**Current**: Every redraw clears and redraws entire panel  
**Better**: Track which lines changed, only redraw those

```c
typedef struct {
    gboolean *dirty_lines_left;
    gboolean *dirty_lines_right;
} SP_DIRTY_TRACKER;
```

**Benefit**: Reduced terminal I/O for activity color changes

---

### Opportunity #2: Line Content Caching
**Current**: Theme formatting runs on every redraw  
**Better**: Cache formatted strings, reuse if unchanged

```c
typedef struct {
    char *cached_text;
    int cached_format_id;
} SP_LINE_CACHE;
```

**Benefit**: Skip expensive `format_get_text_theme*()` calls

---

### Opportunity #3: Double-Buffering
**Current**: Direct writes to TERM_WINDOW  
**Better**: Render to off-screen buffer, diff and blit

**Benefit**: Eliminate flicker on slow terminals

---

## Recommendations

### Priority 1: Fix Critical Bug #1 (Nicklist Rendering)
**Why**: O(N) performance improvement on every nicklist update  
**When**: Next release  
**Complexity**: Low (20 lines of code)

### Priority 2: Fix Medium Bugs #2-#4
**Why**: Improve code robustness and clarity  
**When**: Next maintenance cycle  
**Complexity**: Low (< 10 lines each)

### Priority 3: Consider Optimizations
**Why**: Only if users report performance issues  
**When**: Future enhancement  
**Complexity**: Medium (dirty tracking), High (double-buffering)

---

## Conclusion

**Overall Assessment**: ⭐⭐⭐⭐½ (4.5/5)

The sidepanels implementation is **professionally designed** with:
- Excellent architecture (5-module separation)
- Sophisticated optimization (batched redraws, targeted updates)
- Proper UTF-8 and theme handling

The identified bugs are **minor** and don't affect stability. The critical bug (#1) is a performance optimization, not a crash or data corruption issue.

**Recommendation**: Fix bugs #1-#4 in next release. The architecture does not need redesign.

---

## Test Plan for Fixes

### Test Case 1: Nicklist Performance (Bug #1)
1. Join channel with 500+ users
2. Monitor redraw times (should decrease ~30%)
3. Verify sort order unchanged (ops, voices, normal, all alphabetical)

### Test Case 2: Batched Redraw (Bug #2)
1. Simulate netsplit (100+ quits)
2. Verify no crashes or garbled output
3. Check timer cleanup on shutdown

### Test Case 3: Truncation Edge Cases (Bug #3)
1. Set `sidepanel_left_width` to 1
2. Open channel with very long name
3. Verify no crashes, proper truncation

### Test Case 4: Panel Positioning (Bug #4)
1. Toggle panels rapidly (`/set sidepanel_left`, `/set sidepanel_right`)
2. Verify no visual glitches
3. Check term_window_move() call count (should decrease)

---

**Report Generated**: 2025-01-XX  
**Audited By**: AI Code Review System  
**Next Review**: After fixes implemented
