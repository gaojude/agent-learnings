# iOS Chinese IME Input Issue - Uncontrolled Input Solution

**Date:** January 18, 2026  
**Project:** FurryTogether (Next.js 15 / React 19)  
**Problem:** iOS Chinese input method (26-key keyboard) produces garbled text in controlled React inputs

## Problem Description

When using iOS Chinese input with 26-key keyboard layout in React controlled inputs, typing pinyin like "daiyang" resulted in garbled output like "daaiai yai yaai yaai yanai yanai yangai yang".

### Root Cause

React controlled inputs re-render and update state on every keystroke, which interferes with IME (Input Method Editor) composition process. The IME needs to maintain internal state while converting pinyin to Chinese characters, but React's controlled input pattern disrupts this process by forcing the input value to match React state on every change.

### Initial Failed Approach: IME Composition Events

First attempt used `compositionstart`, `compositionupdate`, and `compositionend` events:

```tsx
const [isComposing, setIsComposing] = useState(false);

const handleCompositionStart = () => setIsComposing(true);
const handleCompositionEnd = (e: CompositionEvent) => {
  setIsComposing(false);
  setSearchQuery(e.currentTarget.value);
};

<input
  value={searchQuery}
  onChange={(e) => !isComposing && setSearchQuery(e.target.value)}
  onCompositionStart={handleCompositionStart}
  onCompositionEnd={handleCompositionEnd}
/>
```

**Result:** Still unreliable on iOS, especially with 26-key layout.

## Solution: Uncontrolled Inputs with Refs

Convert from controlled to uncontrolled input pattern:

### Implementation

```tsx
const searchInputRef = useRef<HTMLInputElement>(null);
const searchTimeoutRef = useRef<NodeJS.Timeout | null>(null);
const [searchQuery, setSearchQuery] = useState('');

const handleSearchInput = (e: React.ChangeEvent<HTMLInputElement>) => {
  const value = e.target.value;
  
  // Clear previous timeout
  if (searchTimeoutRef.current) {
    clearTimeout(searchTimeoutRef.current);
  }
  
  // Debounce search updates
  searchTimeoutRef.current = setTimeout(() => {
    if (value !== searchQuery) {
      setCurrentPage(1);
    }
    startTransition(() => {
      setSearchQuery(value);
    });
  }, 300);
};

const clearSearch = () => {
  if (searchInputRef.current) {
    searchInputRef.current.value = '';
  }
  setSearchQuery('');
  setCurrentPage(1);
};

<input
  ref={searchInputRef}
  type="text"
  defaultValue=""
  onChange={handleSearchInput}
  placeholder="Search..."
/>
```

### Key Changes

1. **Uncontrolled Input**: Use `ref` instead of `value` prop
2. **defaultValue**: Set initial value once, don't control it
3. **Debouncing**: 300ms delay before updating React state
4. **Direct DOM Manipulation**: Clear input via `ref.current.value = ''`

### Why This Works

- **No Re-renders During Typing**: Input value is managed by DOM, not React
- **IME Can Complete**: Input method editor maintains its own state without interference
- **Debounced State Updates**: React state only updates after typing pause, not on every keystroke
- **Better Performance**: Fewer re-renders, smoother UX overall

### Trade-offs

**Pros:**
- Fixes iOS Chinese input completely
- Better performance (fewer re-renders)
- Smoother typing experience
- Works with all IME input methods

**Cons:**
- Input value and React state can temporarily diverge
- Need refs to clear/manipulate input
- Slightly more complex state management

## When to Use

Use uncontrolled inputs with refs when:
- Supporting IME input methods (Chinese, Japanese, Korean, etc.)
- Performance matters (high-frequency updates)
- Input validation happens after typing pause, not real-time
- The input value doesn't need to be controlled externally

Stick with controlled inputs when:
- You need to programmatically set/transform input value on every keystroke
- Real-time validation is required
- Input value must always match React state exactly

## Related Resources

- [React Uncontrolled Components](https://react.dev/reference/react-dom/components/input#controlling-an-input-with-a-state-variable)
- [MDN: Composition Events](https://developer.mozilla.org/en-US/docs/Web/API/CompositionEvent)
- [React 19 Migration Guide](https://react.dev/blog/2024/04/25/react-19)

## Files Modified

- `app/home-client.tsx` - Homepage search bar
- `app/admin/page.tsx` - Admin pending/approved search inputs
