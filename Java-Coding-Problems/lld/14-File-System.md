# In-Memory File System (LLD)

## Problem

Design and build an in-memory file system in Java. The system stores directories and files in a tree, all in memory (no real disk access). It must support:

- `mkdir(path)` — create a directory, including any missing parent directories along the path.
- `addFile(path, content)` / `writeFile(path, content)` — create a file at a path, or overwrite its content if it already exists.
- `readFile(path)` — return the content of a file.
- `ls(path)` — list the names of a directory's direct children, or a single file's own name if the path points to a file. Names come back sorted.
- `rm(path)` — delete a file, or a directory and everything inside it.
- `find(name)` — search the whole tree for files or directories matching a given name, and return all matching paths.

Paths look like `/a/b/c`, similar to a Unix file system. The root is `/`.

This is a strong fit for the **Composite design pattern**. A file system is a tree where some nodes are simple leaves (files) and some nodes are containers that hold other nodes (directories). The Composite pattern lets client code treat a single file and a whole directory the same way, through one common interface, without checking "is this a file or a directory?" everywhere.

## Requirements & Clarifying Questions

An interviewer expects you to clarify scope before writing code. Here are the key questions, with the assumptions we use for this design.

**Functional requirements**

1. The file system is a tree. The root directory is `/`. Every other node has exactly one parent.
2. A path like `/a/b/c` names a chain of directories, ending in either a directory or a file.
3. `mkdir` creates all missing directories along a path (like `mkdir -p` in Unix).
4. `addFile` creates a new file with given content. `writeFile` also updates the content of a file that already exists.
5. `readFile` returns the content of a file, or fails clearly if the path is a directory or does not exist.
6. `ls` lists a directory's direct children only, sorted by name, not the whole subtree.
7. `rm` deletes a file, or a directory with all its contents (recursive delete).
8. `find(name)` walks the whole tree and returns every path whose final segment matches `name`.

**Clarifying questions to ask the interviewer**

- Do we need real persistence (writing to actual disk), or is in-memory storage enough? *Assume: in-memory only. This is stated in the problem, but always confirm it, since "file system" can trigger assumptions about I/O.*
- What content type does a file hold — raw bytes, or plain text? *Assume: a `String` for content, to keep the demo simple. Mention that swapping to `byte[]` is a small change, isolated to the `File` class.*
- Should `mkdir` fail if the directory already exists? *Assume: no, it is idempotent, matching `mkdir -p`. We will note the strict version as a follow-up.*
- Can two children of the same directory share a name? *Assume: no. A directory cannot have two children with the same name, even if one is a file and one is a directory. This mirrors real file systems.*
- Is `find` a simple name match, or does it need wildcards/regex? *Assume: exact name match for the base design, with a follow-up on pattern matching.*
- Do we need thread-safety for concurrent reads and writes? *Assume: yes, discussed as a design point, since a real file system serves many callers at once.*

**Non-functional requirements**

- Adding a new kind of node (for example, a symbolic link) should not force changes to code that already works with files and directories. This is the Open/Closed Principle.
- Client code that walks the tree (like `ls` or `find`) must not need `instanceof` checks to tell files and directories apart.

## Design / Approach

### Why the Composite pattern fits

A file system tree has two kinds of nodes. A **file** is a leaf: it holds content, and it has no children. A **directory** is a container: it holds no content of its own, but it holds a list of child nodes, and each child can itself be a file or another directory.

Without the Composite pattern, code that walks the tree would need `instanceof File` and `instanceof Directory` checks everywhere, plus casting. For example, computing the total size of a path would need: "if it is a file, return its size; if it is a directory, loop over children and check each one's type before recursing." This logic gets copied into every method that walks the tree, and adding a third node type (like a symbolic link) means finding and updating every one of those `instanceof` chains.

The Composite pattern solves this with one common type, here called `FileSystemNode`, that both `File` and `Directory` implement. Every operation that makes sense for "any node in the tree" (get name, get size, print) is declared once on `FileSystemNode`. `File` implements these operations for a single leaf. `Directory` implements the same operations by **delegating to its children** — for example, a directory's size is the sum of its children's sizes, computed by calling `getSize()` on each child, without caring whether that child is itself a file or another directory. This is the core trick of Composite: a container implements the same interface as a leaf, and fulfills it by recursing into its own children.

Client code (like `ls`, `find`, or `rm`) then only ever talks to `FileSystemNode`. It does not need to know, at each step, whether it is looking at a file or a directory, except at the one point where directory-specific behavior (like "add a child") is genuinely needed. That directory-only behavior lives on `Directory`, not on the shared interface, so `File` is never forced to implement a meaningless "add child" method.

### Key design decisions

1. **Composite pattern.** `FileSystemNode` is an abstract base with shared fields (`name`, `parent`) and shared/abstract methods (`getSize()`, `isDirectory()`, `find(...)`). `File` (leaf) holds content and returns its own byte length as size. `Directory` (composite) holds a sorted map of children and returns the sum of children's sizes.
2. **Path parsing lives in one place.** A `PathResolver` (or a static helper) splits a string like `/a/b/c` into segments and walks the tree from the root, one segment at a time. Every public operation (`mkdir`, `readFile`, `ls`, `rm`) reuses this same walk, instead of each method re-parsing paths its own way. This follows the Single Responsibility Principle: one class owns "how do we turn a path string into a node."
3. **Facade over the tree.** `InMemoryFileSystem` is the single entry point client code calls (`mkdir`, `addFile`, `readFile`, `ls`, `rm`, `find`). Internally, it holds the root `Directory` and uses `PathResolver` to reach any node. This keeps the tree classes (`File`, `Directory`) focused only on tree structure, not on path-string parsing.
4. **Sorted children.** `Directory` stores children in a `TreeMap<String, FileSystemNode>`, keyed by name. This gives sorted iteration order for free, which `ls` needs, and O(log n) lookup by name.
5. **No duplicate names in one directory.** `Directory.addChild` throws if a child with that name already exists, whether the existing one is a file or a directory. This matches real file systems and is enforced in one place.
6. **Exceptions for error paths.** Custom unchecked exceptions (`PathNotFoundException`, `PathAlreadyExistsException`, `NotADirectoryException`, `NotAFileException`) make failure cases explicit and let interview demo code show clear error messages, instead of returning `null` and forcing every caller to null-check.
7. **Thread-safety.** Each `Directory` guards its children map with its own lock, so operations on unrelated subtrees do not block each other. This is discussed in detail below.

### ASCII sketch

```
                    ┌───────────────────────────┐
                    │  <<abstract>>               │
                    │  FileSystemNode             │
                    │  - name: String             │
                    │  - parent: Directory        │
                    │  + getName(): String        │
                    │  + getPath(): String        │
                    │  + getSize(): long          │
                    │  + isDirectory(): boolean   │
                    │  + find(name, out): void    │
                    └──────────────┬──────────────┘
                                   │
                 ┌─────────────────┴─────────────────┐
                 ▼                                    ▼
      ┌───────────────────────┐          ┌─────────────────────────────┐
      │  File   (Leaf)         │          │  Directory   (Composite)     │
      │  - content: String     │          │  - children: TreeMap<String, │
      │  + getContent()        │          │        FileSystemNode>       │
      │  + setContent(s)       │          │  + addChild(node)            │
      │  + getSize() = len(c)  │          │  + removeChild(name)         │
      │                        │          │  + getChild(name)            │
      │                        │          │  + listChildren(): sorted    │
      │                        │          │  + getSize() = sum(children) │
      └───────────────────────┘          └─────────────────────────────┘

                    ┌───────────────────────────┐
                    │  PathResolver               │  (helper: path string -> node)
                    │  + resolve(root, path)      │
                    │  + resolveParent(root, path)│
                    └──────────────┬──────────────┘
                                   │ used by
                    ┌──────────────▼──────────────┐
                    │  InMemoryFileSystem           │  (Facade)
                    │  - root: Directory             │
                    │  + mkdir(path)                 │
                    │  + addFile(path, content)      │
                    │  + writeFile(path, content)    │
                    │  + readFile(path): String      │
                    │  + ls(path): List<String>      │
                    │  + rm(path)                    │
                    │  + find(name): List<String>    │
                    └────────────────────────────────┘
```

Patterns used: **Composite** (`FileSystemNode` base with `File` leaf and `Directory` composite — the central pattern of this problem), **Facade** (`InMemoryFileSystem` gives client code one simple entry point over the tree and the path resolver), and **Template-ish delegation** (`Directory.getSize()` and `Directory.find()` delegate to children's own implementations, so the composite never needs to know what kind of node each child is).

## Java Solution

```java
import java.util.*;
import java.util.concurrent.locks.ReentrantReadWriteLock;

// ---------- Exceptions ----------
class PathNotFoundException extends RuntimeException {
    PathNotFoundException(String path) { super("Path not found: " + path); }
}
class PathAlreadyExistsException extends RuntimeException {
    PathAlreadyExistsException(String path) { super("Path already exists: " + path); }
}
class NotADirectoryException extends RuntimeException {
    NotADirectoryException(String path) { super("Not a directory: " + path); }
}
class NotAFileException extends RuntimeException {
    NotAFileException(String path) { super("Not a file: " + path); }
}

// ---------- FileSystemNode: the Composite base ----------
public abstract class FileSystemNode {
    protected String name;
    protected Directory parent;

    protected FileSystemNode(String name, Directory parent) {
        this.name = name;
        this.parent = parent;
    }

    public String getName() { return name; }
    public Directory getParent() { return parent; }

    // Every node knows its own full path, by walking up to the root.
    public String getPath() {
        if (parent == null) return "/"; // this node IS the root
        String parentPath = parent.getPath();
        return parentPath.equals("/") ? "/" + name : parentPath + "/" + name;
    }

    public abstract boolean isDirectory();
    public abstract long getSize();

    // Recursive search: each node checks itself, and (for directories) asks
    // its children to do the same. A File's override is trivial (no children).
    public abstract void find(String targetName, List<String> results);
}

// ---------- File: the Leaf ----------
public final class File extends FileSystemNode {
    private String content;

    public File(String name, Directory parent, String content) {
        super(name, parent);
        this.content = content == null ? "" : content;
    }

    public String getContent() { return content; }
    public void setContent(String content) { this.content = content == null ? "" : content; }

    @Override
    public boolean isDirectory() { return false; }

    @Override
    public long getSize() { return content.length(); } // characters, as a stand-in for bytes

    @Override
    public void find(String targetName, List<String> results) {
        if (this.name.equals(targetName)) {
            results.add(getPath());
        }
        // A file has no children, so the search stops here.
    }
}

// ---------- Directory: the Composite ----------
public final class Directory extends FileSystemNode {
    // TreeMap keeps children sorted by name, which "ls" needs anyway.
    private final TreeMap<String, FileSystemNode> children = new TreeMap<>();
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public Directory(String name, Directory parent) {
        super(name, parent);
    }

    @Override
    public boolean isDirectory() { return true; }

    @Override
    public long getSize() {
        lock.readLock().lock();
        try {
            long total = 0;
            for (FileSystemNode child : children.values()) {
                total += child.getSize(); // works for File or Directory, no type check needed
            }
            return total;
        } finally {
            lock.readLock().unlock();
        }
    }

    @Override
    public void find(String targetName, List<String> results) {
        lock.readLock().lock();
        try {
            if (this.name.equals(targetName)) {
                results.add(getPath());
            }
            for (FileSystemNode child : children.values()) {
                child.find(targetName, results); // delegate; child decides how to search itself
            }
        } finally {
            lock.readLock().unlock();
        }
    }

    public void addChild(FileSystemNode node) {
        lock.writeLock().lock();
        try {
            if (children.containsKey(node.getName())) {
                throw new PathAlreadyExistsException(getPath() + "/" + node.getName());
            }
            children.put(node.getName(), node);
        } finally {
            lock.writeLock().unlock();
        }
    }

    public void removeChild(String childName) {
        lock.writeLock().lock();
        try {
            if (children.remove(childName) == null) {
                throw new PathNotFoundException(getPath() + "/" + childName);
            }
        } finally {
            lock.writeLock().unlock();
        }
    }

    public FileSystemNode getChild(String childName) {
        lock.readLock().lock();
        try {
            return children.get(childName);
        } finally {
            lock.readLock().unlock();
        }
    }

    // Sorted names of direct children only (not the whole subtree).
    public List<String> listChildNames() {
        lock.readLock().lock();
        try {
            return new ArrayList<>(children.keySet()); // TreeMap keySet is already sorted
        } finally {
            lock.readLock().unlock();
        }
    }
}

// ---------- PathResolver: turns "/a/b/c" into tree walks ----------
public final class PathResolver {

    private static List<String> splitSegments(String path) {
        if (path == null || path.isEmpty() || path.charAt(0) != '/') {
            throw new IllegalArgumentException("Path must be absolute, starting with '/': " + path);
        }
        List<String> segments = new ArrayList<>();
        for (String part : path.split("/")) {
            if (!part.isEmpty()) segments.add(part);
        }
        return segments; // e.g. "/a/b/c" -> ["a", "b", "c"]; "/" -> []
    }

    // Resolves the full path to its node. Throws if any segment along the
    // way is missing, or if a non-final segment is a File (cannot descend into a file).
    public static FileSystemNode resolve(Directory root, String path) {
        List<String> segments = splitSegments(path);
        FileSystemNode current = root;
        for (String segment : segments) {
            if (!current.isDirectory()) {
                throw new NotADirectoryException(current.getPath());
            }
            FileSystemNode next = ((Directory) current).getChild(segment);
            if (next == null) throw new PathNotFoundException(path);
            current = next;
        }
        return current;
    }

    // Resolves the PARENT directory of the last segment, creating nothing.
    // Useful for "add a new child named X under this parent".
    public static Directory resolveParent(Directory root, String path) {
        List<String> segments = splitSegments(path);
        if (segments.isEmpty()) {
            throw new IllegalArgumentException("Path has no parent: " + path);
        }
        Directory current = root;
        for (int i = 0; i < segments.size() - 1; i++) {
            FileSystemNode next = current.getChild(segments.get(i));
            if (next == null) throw new PathNotFoundException(path);
            if (!next.isDirectory()) throw new NotADirectoryException(next.getPath());
            current = (Directory) next;
        }
        return current;
    }

    public static String lastSegment(String path) {
        List<String> segments = splitSegments(path);
        if (segments.isEmpty()) throw new IllegalArgumentException("Path has no name: " + path);
        return segments.get(segments.size() - 1);
    }
}

// ---------- InMemoryFileSystem: the Facade client code talks to ----------
public final class InMemoryFileSystem {
    private final Directory root = new Directory("", null); // name "" ; getPath() returns "/"

    // Creates every missing directory along the path (like "mkdir -p").
    public void mkdir(String path) {
        List<String> segments = splitPathPublic(path);
        Directory current = root;
        for (String segment : segments) {
            FileSystemNode next = current.getChild(segment);
            if (next == null) {
                Directory newDir = new Directory(segment, current);
                current.addChild(newDir);
                current = newDir;
            } else if (next.isDirectory()) {
                current = (Directory) next; // already exists: fine, keep walking (idempotent)
            } else {
                throw new NotADirectoryException(next.getPath());
            }
        }
    }

    public void addFile(String path, String content) {
        Directory parent = PathResolver.resolveParent(root, path);
        String fileName = PathResolver.lastSegment(path);
        if (parent.getChild(fileName) != null) {
            throw new PathAlreadyExistsException(path);
        }
        parent.addChild(new File(fileName, parent, content));
    }

    // Overwrites content if the file exists; creates it otherwise.
    public void writeFile(String path, String content) {
        Directory parent = PathResolver.resolveParent(root, path);
        String fileName = PathResolver.lastSegment(path);
        FileSystemNode existing = parent.getChild(fileName);
        if (existing == null) {
            parent.addChild(new File(fileName, parent, content));
        } else if (existing instanceof File) {
            ((File) existing).setContent(content);
        } else {
            throw new NotAFileException(path);
        }
    }

    public String readFile(String path) {
        FileSystemNode node = PathResolver.resolve(root, path);
        if (!(node instanceof File)) throw new NotAFileException(path);
        return ((File) node).getContent();
    }

    // Lists a directory's direct children (sorted), or a single file's own name.
    public List<String> ls(String path) {
        FileSystemNode node = path.equals("/") ? root : PathResolver.resolve(root, path);
        if (node.isDirectory()) {
            return ((Directory) node).listChildNames();
        }
        return List.of(node.getName());
    }

    public void rm(String path) {
        Directory parent = PathResolver.resolveParent(root, path);
        String name = PathResolver.lastSegment(path);
        parent.removeChild(name); // Directory holds the only reference; child (and its subtree) is now unreachable and GC-eligible
    }

    // Returns full paths of every node (file or directory) whose name matches.
    public List<String> find(String name) {
        List<String> results = new ArrayList<>();
        root.find(name, results);
        return results;
    }

    public long getSize(String path) {
        FileSystemNode node = path.equals("/") ? root : PathResolver.resolve(root, path);
        return node.getSize();
    }

    private List<String> splitPathPublic(String path) {
        // Reuses the same parsing rule as PathResolver, kept private here
        // only to avoid exposing an extra public method on PathResolver.
        if (path.equals("/")) return List.of();
        List<String> segments = new ArrayList<>();
        for (String part : path.split("/")) {
            if (!part.isEmpty()) segments.add(part);
        }
        return segments;
    }
}
```

### Example usage

```java
public class Demo {
    public static void main(String[] args) {
        InMemoryFileSystem fs = new InMemoryFileSystem();

        fs.mkdir("/home/user/docs");          // creates home, user, docs in one call
        fs.addFile("/home/user/docs/todo.txt", "buy milk");
        fs.addFile("/home/user/notes.txt", "meeting at 3pm");

        System.out.println(fs.ls("/home/user"));
        // -> [docs, notes.txt]   (sorted, direct children only)

        System.out.println(fs.readFile("/home/user/docs/todo.txt"));
        // -> buy milk

        fs.writeFile("/home/user/docs/todo.txt", "buy milk and eggs");
        System.out.println(fs.readFile("/home/user/docs/todo.txt"));
        // -> buy milk and eggs

        fs.mkdir("/home/user/docs/backup");
        fs.addFile("/home/user/docs/backup/todo.txt", "old copy");

        System.out.println(fs.find("todo.txt"));
        // -> [/home/user/docs/todo.txt, /home/user/docs/backup/todo.txt]

        System.out.println(fs.getSize("/home/user"));
        // -> sum of sizes of notes.txt, docs/todo.txt, docs/backup/todo.txt

        fs.rm("/home/user/docs/backup");      // deletes the directory and its file inside
        System.out.println(fs.find("todo.txt"));
        // -> [/home/user/docs/todo.txt]   (backup copy is gone)
    }
}
```

## How It Works

1. **`FileSystemNode` is the Composite's common interface.** Both `File` and `Directory` extend it, and both must implement `getSize()` and `find(...)`. Client code and even `Directory` itself only ever call these methods through the `FileSystemNode` reference type. Neither `Directory.getSize()` nor `Directory.find()` contains an `instanceof` check; they simply call `child.getSize()` or `child.find(...)` and trust that each child, whatever kind it is, does the right thing. This is the heart of the Composite pattern: uniform treatment of leaves and containers.
2. **`File` is the leaf.** It has no children, so its `getSize()` returns its own content length directly, and its `find()` only checks its own name, with no recursion. A leaf is where the recursion in a composite tree naturally stops.
3. **`Directory` is the composite.** It holds children in a `TreeMap<String, FileSystemNode>`. Its `getSize()` loops over children and sums their sizes; its `find()` checks itself, then asks every child to search itself. Because `children` is typed as `FileSystemNode`, a `Directory` never needs to know whether a given child is a `File` or another `Directory` when doing these generic operations. It only needs to know the concrete type for directory-only actions like `addChild`, which makes sense, since only a directory can hold new children.
4. **`PathResolver` centralizes path parsing.** Every operation that needs to reach a node from a string path (`mkdir`, `addFile`, `readFile`, `ls`, `rm`) goes through `PathResolver.resolve` or `PathResolver.resolveParent`. This means the rule "a path is split on `/`, empty segments are ignored, and a non-final segment must be a directory" is written once, not five times. If that rule ever changes (say, to support `.` and `..`), only `PathResolver` needs an update.
5. **`mkdir` is idempotent by design.** Walking the path, if a segment already exists as a directory, `mkdir` just moves into it and continues; it only creates a new `Directory` node when a segment is missing. If a segment exists but is a `File`, it throws `NotADirectoryException`, since you cannot make a directory "inside" a file.
6. **`rm` deletes a whole subtree by removing one reference.** `Directory.removeChild` just takes the target name out of the parent's map. Since Java is garbage collected, and nothing else holds a reference to the removed subtree (each node's `parent` pointer does not keep the parent tree alive on its own, and no external references exist in this design), the whole subtree — the directory and everything inside it — becomes unreachable and gets garbage collected. There is no need to write a manual recursive-delete loop.
7. **`find` returns full paths, computed by walking up `parent` pointers.** `getPath()` on any node walks up through `parent` references, building the path from the root down. This is why every node stores a reference to its `Directory parent` at construction time, set once and never changed in this design (moving a node to a new parent is listed as a follow-up).
8. **`InMemoryFileSystem` is the Facade.** It is the only class most client code touches. It owns the root `Directory` and coordinates `PathResolver` calls with tree mutations (`addChild`, `removeChild`). This keeps `File` and `Directory` focused purely on tree structure and Composite behavior, and keeps path-string handling out of the tree classes entirely — a clean separation of concerns.

## How to Extend (Follow-ups)

Interviewers like to ask "what if we needed X?" after the base design works. Have these ready.

- **File size limits and quotas.** Add a `maxSizeBytes` field to `File`, checked in `setContent`, and a `quotaBytes` field to `Directory`, checked before `addChild` or `setContent` succeeds, by calling the existing `getSize()` up the parent chain. Since `getSize()` already sums recursively, quota checks reuse it directly instead of needing new traversal code. This is a good example of the Composite pattern paying off again: the same recursive `getSize()` used for reporting is reused for enforcement.
- **Permissions.** Add an `owner` and a `Set<Permission>` (read, write, execute) to `FileSystemNode`, checked at the start of every operation in `InMemoryFileSystem`, given a "current user" parameter threaded through the API. A directory without execute permission should block resolving paths through it, even if the caller somehow knows a child's name.
- **Symbolic links.** Add a third `FileSystemNode` subtype, `SymbolicLink`, that stores a target path instead of content or children. `PathResolver.resolve` needs one addition: when it encounters a `SymbolicLink` mid-path, it re-resolves from the link's target path before continuing. This is exactly the benefit of Composite: `SymbolicLink` is a new leaf-like type, and existing code that calls `getSize()` or `find()` on `FileSystemNode` does not need to change, only `PathResolver` needs a small, localized update, plus a cycle-detection guard (a visited-set or a maximum hop count) to avoid infinite loops from a link pointing at itself or at an ancestor.
- **Move / rename.** Add a `move(oldPath, newPath)` method: resolve the node at `oldPath`, resolve the new parent directory, call `oldParent.removeChild(name)` then `newParent.addChild(node)` with an updated `name` and `parent` field on the node. Because `parent` is currently set only at construction, this follow-up needs `File`/`Directory` to expose a package-visible setter for `parent`, guarded the same way `addChild`/`removeChild` are guarded.
- **How a real file system differs (inodes and blocks), briefly.** A real file system (like ext4 or NTFS) does not store a name-to-content tree directly. Instead, each file or directory has an **inode**: a fixed-size record holding metadata (size, permissions, timestamps, and pointers to data blocks on disk), but not the file's name. A directory is itself just a special file whose content is a list of (name, inode number) pairs. Storage is split into fixed-size **blocks** (for example, 4 KB each); a file's data is spread across as many blocks as it needs, and the inode holds pointers to those blocks (directly, or through indirect pointer blocks for large files). This separation is why a "hard link" is possible on a real file system: two different names, in two different directories, can point to the same inode. Our in-memory design skips this layer, since one Java object reference already plays the role of "pointer to the actual data," but it is worth mentioning this gap if an interviewer probes deeper into real-world file systems.
- **Pattern-based search.** Change `find(String name, ...)` to `find(Predicate<FileSystemNode> matcher, ...)`, so callers can search by size, by extension, or by a name pattern (glob or regex), without adding a new method for every kind of search. The recursive traversal in `Directory.find` does not need to change at all, only the check `this.name.equals(targetName)` becomes `matcher.test(this)`.

## Complexity & Thread-Safety Notes

- **`mkdir(path)`:** O(d) where d is the depth of the path (number of segments), since it walks or creates one directory per segment.
- **`addFile` / `writeFile` / `readFile` / `rm`:** O(d + log k), where d is path depth (to resolve the parent directory) and k is the number of children in the relevant directory (`TreeMap` lookup/insert/remove is O(log k)).
- **`ls(path)`:** O(d + k), where d is the depth to resolve the path, and k is the number of direct children, since `listChildNames()` copies the sorted key set.
- **`find(name)`:** O(n), where n is the total number of nodes in the tree, since in the worst case (name not found, or found only at the deepest leaf) every node must be visited once.
- **`getSize(path)`:** O(m) where m is the number of nodes in the subtree rooted at that path, since the sum is computed by visiting every descendant.
- **Space:** O(n) for n total nodes, plus O(d) recursion stack depth for `getSize` and `find` on deeply nested paths.
- **Thread-safety summary:**
    - Each `Directory` has its own `ReentrantReadWriteLock`, guarding only its own `children` map. This is **fine-grained locking**: an operation inside `/home/alice` does not block an unrelated operation inside `/home/bob`, since they lock different `Directory` instances.
    - Reads (`getChild`, `listChildNames`, the read side of `getSize`/`find`) take the **read lock**, so many readers can proceed at once on the same directory. Writes (`addChild`, `removeChild`) take the **write lock**, which excludes both other writers and readers on that directory while it runs.
    - This design is safe for the same reason a real file system is safe under concurrent access to different folders: a lock is scoped to "this directory's own child list," not to the whole tree, so unrelated parts of the tree do not contend.
    - **What this does NOT protect against:** a multi-step client operation, like "check if a file exists, then create it if not," done as two separate `InMemoryFileSystem` calls, is not atomic as a whole, even though each individual call is thread-safe internally. Two threads could both see "does not exist" and both then call `addFile`; one would get a `PathAlreadyExistsException`. If atomic check-and-create is required, add a single method that does both steps while holding the target directory's write lock throughout, rather than composing two separately-locked calls.
    - `File.content` is a plain `String` field with no lock of its own in this design. Since `writeFile` replaces the field's value only while the parent `Directory`'s write lock is not actually protecting `File` internals (it protects the parent's child map, not sibling files' content), a concurrent `readFile` and `writeFile` on the exact same file could, in theory, interleave unsafely at the field level, though `String` immutability in Java means a reader will always see either the old or the new complete string, never a partially-written one. For stricter guarantees, wrap `content` access in `File` with its own lock, or make the field `volatile` to guarantee visibility of the latest full reference across threads.

## Interview Tips & Common Mistakes

- **Say why Composite fits before writing code.** Something like: "files and directories need to be treated the same way by code that walks the tree, like computing size or searching by name, but a directory also needs extra behavior — holding children — that a file does not. I will put the shared behavior on a common `FileSystemNode` type, and let `Directory` implement it by delegating to its children." This shows you understand the pattern's purpose, not just its class names.
- **A common mistake:** putting `List<FileSystemNode> children` on the base `FileSystemNode` class "just in case," even for `File`. This breaks the point of Composite: a leaf should not carry container-only state. Keep `children` only on `Directory`.
- **Another common mistake:** writing `ls`, `find`, or `getSize` with `if (node instanceof File) ... else if (node instanceof Directory) ...`. If you see yourself writing this, stop — it means the shared behavior is not actually on the base class yet. The whole point of Composite is that this kind of branch should not exist in client code or in `Directory`'s own methods.
- **Do not forget duplicate-name checks.** A frequent bug: `addChild` silently overwrites an existing child with the same name. Real file systems reject this (`mkdir` on an existing path is fine, but creating a file with the same name as an existing directory is not). Make the rule explicit and enforce it in exactly one place (`Directory.addChild`).
- **Be ready to explain why `rm` does not need a manual recursive delete.** Some candidates write a recursive `deleteAll(node)` helper. It is not wrong, but it is unnecessary here: removing the one reference from the parent's map is enough, since Java's garbage collector reclaims the now-unreachable subtree. Mentioning this shows you understand both the pattern and the language's memory model.
- **If asked "how would you make this a real, persisted file system,"** bring up the inode/block model from the follow-ups section: names live in directory entries, actual data lives in blocks referenced by an inode, and multiple names can point at the same inode (hard links) — something not directly possible when a name maps straight to a Java object with a single conceptual owner, as in this in-memory design.
- **If asked about scaling this to a distributed file system** (like HDFS or S3), mention that the tree/metadata layer (names, directory structure) and the data layer (actual bytes) are usually split into separate services: a metadata service tracks the tree shape (similar to this `Directory`/`File` structure, but backed by a database), while a separate storage layer holds the actual bytes, often replicated across machines. This separation is what lets a distributed file system scale metadata operations and data storage independently.
