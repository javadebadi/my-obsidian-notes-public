**Physical storage hardware**

There are two kinds of storage hardware:

- HDD
- SSD

Both store data as raw blocks of 0s and 1s.

---

**Software abstractions**

**Filesystem** — software that maintains an index mapping human-readable units (files, folders) to physical blocks. This is what every OS provides and what you interact with every day.

**Block storage** — skips the filesystem entirely. The application talks directly to raw blocks. Used by software (like databases) that manages its own data layout for performance reasons.

**Object storage** — the unit of storage is the object itself, not blocks or files. The system handles how and where the object is physically stored — potentially across many devices. You just store and retrieve the whole object. Examples: S3, GCS.

---

**Key insight**

Filesystem and block storage are abstractions on top of hardware. Object storage goes further — it abstracts away the hardware entirely, even across multiple physical devices. The object is the unit, not the block.