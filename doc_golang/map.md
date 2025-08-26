# sync.Mutex 代码解读

# go 版本: 1.22.2

# 设计思想
- sync.map 底层其实是两个 map，一个 read map，一个 dirty map，read map 并发读安全，所有读操作优先read map，所有写直接操作 dirty map，read map 和 dirty map 在需要时间会进行数据同步
- 标记删除，元素不会在调用删除时立刻删除，而是增加标记，然后直到 dirty map 提升时删除

# 关键结构
```go
type Map struct {
	mu Mutex

    // read map，读并发安全，更新时需要对 mu 加锁
	read atomic.Pointer[readOnly]

	// dirty map
	dirty map[any]*entry

    // read map 最后一次更新后，load 缓存失效次数，次数够多时将 dirty map 提升为 read map
	misses int
}

// readOnly 是存在 Map.read 中永远不会被修改的结构
type readOnly struct {
	m       map[any]*entry
	amended bool // 如果 dirty map 包含 read map 中不存在的元素则为 true
}

// expunged 是一个固定的标记，表示一个元素已经被逻辑删除
var expunged = new(any)

// entry 条目是映射中与特定键相对应的槽
type entry struct {
    // p 是指向一个值的指针
	//
	// 如果 p == nil, 则 entry 已经被删除，此时 m.dirty == nil 或者 m.dirty[key] 是 entry
	// 如果 p == expunged, 则 entry 已经被删除, 此时 m.dirty != nil, 且 m.dirty 不存在 entry
	// 否则, entry 有效，存在于 m.read.m[key] 并且如果 m.dirty != nil 则同时存在于 m.dirty[key]
	p atomic.Pointer[any]
}
```

# 代码解读
```go
// newEntry 初始化新的 entry
func newEntry(i any) *entry {
	e := &entry{}
	e.p.Store(&i)
	return e
}

// 加载 read map
func (m *Map) loadReadOnly() readOnly {
	if p := m.read.Load(); p != nil {
		return *p
	}
	return readOnly{}
}

// 加载一个元素
func (m *Map) Load(key any) (value any, ok bool) {
	read := m.loadReadOnly()
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()

        // 加锁 && double check
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]

            // 增加 miss 计数，如果过大则将 dirty map 提升成 read map
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}

// load 加载 entry 中的元素，如果 p == nil 或 p == expunged 都表示元素已删除
func (e *entry) load() (value any, ok bool) {
	p := e.p.Load()
	if p == nil || p == expunged {
		return nil, false
	}
	return *p, true
}

// Store 存储一个元素
func (m *Map) Store(key, value any) {
	_, _ = m.Swap(key, value)
}

// tryCompareAndSwap 尝试替换一个值
func (e *entry) tryCompareAndSwap(old, new any) bool {
	p := e.p.Load()
    // 元素被删或者旧值变更就直接失败
	if p == nil || p == expunged || *p != old {
		return false
	}

	nc := new
	for {
        // 如果刚好被并发更新了值就重试直到成功
		if e.p.CompareAndSwap(p, &nc) {
			return true
		}
		p = e.p.Load()
		if p == nil || p == expunged || *p != old {
			return false
		}
	}
}

// unexpungeLocked 删除 expunged 标记
func (e *entry) unexpungeLocked() (wasExpunged bool) {
	return e.p.CompareAndSwap(expunged, nil)
}

// swapLocked 原子性的替换指针
func (e *entry) swapLocked(i *any) *any {
	return e.p.Swap(i)
}

// LoadOrStore
func (m *Map) LoadOrStore(key, value any) (actual any, loaded bool) {
    // 快速路径，无锁查询 read map
	read := m.loadReadOnly()
	if e, ok := read.m[key]; ok {
		actual, loaded, ok := e.tryLoadOrStore(value)
		if ok {
			return actual, loaded
		}
	}

	m.mu.Lock()
	read = m.loadReadOnly()
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			m.dirty[key] = e
		}
		actual, loaded, _ = e.tryLoadOrStore(value)
	} else if e, ok := m.dirty[key]; ok {
		actual, loaded, _ = e.tryLoadOrStore(value)
		m.missLocked()
	} else {
		if !read.amended {
			m.dirtyLocked()
			m.read.Store(&readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
		actual, loaded = value, false
	}
	m.mu.Unlock()

	return actual, loaded
}

// tryLoadOrStore
func (e *entry) tryLoadOrStore(i any) (actual any, loaded, ok bool) {
    // 无锁快速判断
	p := e.p.Load()
	if p == expunged {
		return nil, false, false
	}
	if p != nil {
		return *p, true, true
	}

	ic := i
	for {
		if e.p.CompareAndSwap(nil, &ic) {
			return i, false, true
		}
		p = e.p.Load()
		if p == expunged {
			return nil, false, false
		}
		if p != nil {
			return *p, true, true
		}
	}
}

// LoadAndDelete
func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
	read := m.loadReadOnly()
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
            // 从 dirty map 实际删除
			delete(m.dirty, key)
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if ok {
        // 逻辑删除
		return e.delete()
	}
	return nil, false
}

// delete 标记元素逻辑删除
func (e *entry) delete() (value any, ok bool) {
	for {
		p := e.p.Load()
		if p == nil || p == expunged {
			return nil, false
		}
		if e.p.CompareAndSwap(p, nil) {
			return *p, true
		}
	}
}

// trySwap 快速路径尝试交换
func (e *entry) trySwap(i *any) (*any, bool) {
	for {
		p := e.p.Load()
		if p == expunged {
			return nil, false
		}
		if e.p.CompareAndSwap(p, i) {
			return p, true
		}
	}
}

func (m *Map) Swap(key, value any) (previous any, loaded bool) {
	read := m.loadReadOnly()
	if e, ok := read.m[key]; ok {
		if v, ok := e.trySwap(&value); ok {
			if v == nil {
				return nil, false
			}
			return *v, true
		}
	}

	m.mu.Lock()
	read = m.loadReadOnly()
    // 先从 read map 尝试读一次
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			// 如果原先有 expunged 标记，则表示 dirty map 非空，并需要更新该值到 dirty map
			m.dirty[key] = e
		}
		if v := e.swapLocked(&value); v != nil {
            // 返回旧值
			loaded = true
			previous = *v
		}
	} else if e, ok := m.dirty[key]; ok {
        // 从 dirty map 查询
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else {
		if !read.amended {
            // 如果是插入第一个值到 dirty map，需要标记 read map 不完整
			m.dirtyLocked()
			m.read.Store(&readOnly{m: read.m, amended: true})
		}
        // 直接往 dirty map 插入新值
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
	return previous, loaded
}

// CompareAndSwap
func (m *Map) CompareAndSwap(key, old, new any) bool {
	read := m.loadReadOnly()
	if e, ok := read.m[key]; ok {
		return e.tryCompareAndSwap(old, new)
	} else if !read.amended {
		return false // No existing value for key.
	}

	m.mu.Lock()
	defer m.mu.Unlock()
	read = m.loadReadOnly()
	swapped := false
	if e, ok := read.m[key]; ok {
		swapped = e.tryCompareAndSwap(old, new)
	} else if e, ok := m.dirty[key]; ok {
		swapped = e.tryCompareAndSwap(old, new)
		m.missLocked()
	}
	return swapped
}

// CompareAndDelete
func (m *Map) CompareAndDelete(key, old any) (deleted bool) {
	read := m.loadReadOnly()
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			m.missLocked()
		}
		m.mu.Unlock()
	}
	for ok {
		p := e.p.Load()
		if p == nil || p == expunged || *p != old {
			return false
		}
		if e.p.CompareAndSwap(p, nil) {
			return true
		}
	}
	return false
}

// Range 遍历元素
func (m *Map) Range(f func(key, value any) bool) {
	read := m.loadReadOnly()
	if read.amended {
		m.mu.Lock()
		read = m.loadReadOnly()
		if read.amended {
            // 提升 dirty map 成 read map
			read = readOnly{m: m.dirty}
			copyRead := read
			m.read.Store(&copyRead)
			m.dirty = nil
			m.misses = 0
		}
		m.mu.Unlock()
	}

	for k, e := range read.m {
		v, ok := e.load()
		if !ok {
			continue
		}
		if !f(k, v) {
			break
		}
	}
}

// missLocked 增加 miss 计数，如果过大则将 dirty map 提升成 read map
func (m *Map) missLocked() {
	m.misses++
	if m.misses < len(m.dirty) {
		return
	}
	m.read.Store(&readOnly{m: m.dirty})
	m.dirty = nil
	m.misses = 0
}

// dirtyLocked dirty map 为空时，拷贝 read map 所有未删除元素
func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read := m.loadReadOnly()
	m.dirty = make(map[any]*entry, len(read.m))
	for k, e := range read.m {
		if !e.tryExpungeLocked() {
            // 拷贝 read map 所有未删除元素到 dirty map
			m.dirty[k] = e
		}
	}
}

// tryExpungeLocked
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := e.p.Load()
    // 将 nil 指针标记为 expunged
	for p == nil {
		if e.p.CompareAndSwap(nil, expunged) {
			return true
		}
		p = e.p.Load()
	}
	return p == expunged
}
```