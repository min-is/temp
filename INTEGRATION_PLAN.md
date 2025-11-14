# Docmost → Scribe Integration Plan
*A Comprehensive Strategy for Document Management Integration*

---

## Executive Summary

This plan outlines strategies to integrate Docmost's sophisticated document management capabilities into the Scribe medical education dashboard. After thorough analysis of both codebases, I've identified key features to adopt and multiple implementation routes based on complexity and impact.

---

## Table of Contents

1. [Architecture Comparison](#architecture-comparison)
2. [Key Features to Adopt](#key-features-to-adopt)
3. [Integration Routes](#integration-routes)
4. [Detailed Implementation Plans](#detailed-implementation-plans)
5. [Technical Specifications](#technical-specifications)
6. [Migration Strategy](#migration-strategy)
7. [Timeline & Priorities](#timeline--priorities)

---

## Architecture Comparison

### Docmost Architecture
| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Database ORM** | Kysely (SQL builder) | Type-safe queries with flexibility |
| **Editor** | Tiptap 2.x + Custom Extensions | Rich document editing |
| **Collaboration** | Yjs + Hocuspocus | Real-time collaborative editing |
| **Storage** | Driver pattern (Local/S3) | Abstracted file storage |
| **Page Organization** | Hierarchical tree with fractional indexing | Efficient page ordering |
| **Backend** | NestJS + Fastify | Modular API architecture |
| **State Management** | Jotai + React Query | Atomic state + server cache |

### Scribe Architecture
| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Database ORM** | Prisma | Type-safe ORM with migrations |
| **Editor** | Tiptap 3.x + Basic Extensions | Rich text editing |
| **Collaboration** | None | Single-user editing |
| **Storage** | Vercel Blob (+ R2/S3 support) | Cloud file storage |
| **Page Organization** | Flat structure with slugs | Simple content organization |
| **Backend** | Next.js 15 App Router + Server Actions | Serverless API |
| **State Management** | React Context + SWR | Simple state + server cache |

### Key Differences
- **Docmost**: Workspace → Spaces → Pages (3-level hierarchy)
- **Scribe**: Single-level content organization per type
- **Docmost**: Real-time collaboration with CRDT
- **Scribe**: Static content editing
- **Docmost**: Fractional indexing for ordering
- **Scribe**: No explicit ordering system

---

## Key Features to Adopt

### 🎯 High Priority (Core Document Management)

#### 1. Hierarchical Page Organization
**What**: Parent-child page relationships with unlimited nesting
**Why**: Enables knowledge base structure, documentation trees, and organized content
**Example**: Provider → Documentation → Procedures → Specific Procedure Steps

**Technical Details**:
- Self-referencing `parentId` field
- Recursive queries for breadcrumbs and tree traversal
- Drag-and-drop reordering

#### 2. Fractional Indexing for Page Order
**What**: String-based positioning system (e.g., "a0", "a1", "b0")
**Why**: Reorder pages without updating siblings (O(1) operation)
**Library**: `fractional-indexing-jittered` (already in Docmost)

**Technical Details**:
- Position stored as string, not integer
- New positions calculated between existing positions
- No database updates to sibling records

#### 3. Enhanced Attachment Management
**What**: Comprehensive file metadata, associations, and lifecycle management
**Why**: Better file organization, search, and reuse

**Features**:
- Link attachments to multiple pages
- File metadata (size, mime type, upload date)
- Attachment search by filename/content
- Orphan detection and cleanup

#### 4. Rich Editor Extensions
**What**: Additional Tiptap extensions from Docmost
**Why**: More powerful editing capabilities

**Extensions to Add**:
- **Math**: KaTeX inline/block equations (`TiptapMathInline`, `TiptapMathBlock`)
- **Diagrams**: Excalidraw, Draw.io, Mermaid support
- **Details/Summary**: Collapsible sections
- **Callouts**: Styled info boxes (warning, info, success, etc.)
- **Comments**: Inline comment threads (if collaboration is desired)
- **Search & Replace**: Find/replace in editor
- **Subpages**: List child pages in document
- **Advanced Tables**: Row/column drag-and-drop

#### 5. Page Metadata System
**What**: Comprehensive metadata for every page
**Why**: Better organization, search, and UX

**Metadata Fields**:
- `icon` (emoji or icon identifier)
- `coverPhoto` (header image URL)
- `isLocked` (prevent editing)
- `contributorIds` (track who edited)
- `position` (fractional index)
- `viewCount` (analytics)

---

### 🔄 Medium Priority (Enhanced Features)

#### 6. Storage Driver Pattern
**What**: Abstract storage interface supporting multiple backends
**Why**: Flexibility to switch between Vercel Blob, S3, R2, local storage

**Benefits**:
- Unified interface for all storage operations
- Easy migration between providers
- Testing with local storage

#### 7. Page History/Versioning
**What**: Snapshot history of page changes
**Why**: Recovery, audit trail, comparison

**Implementation**:
- Store snapshots in `PageHistory` table
- Track who, when, and what changed
- Restore to previous version

#### 8. Soft Deletes with Trash
**What**: Recoverable deletion system
**Why**: Prevent accidental data loss

**Features**:
- `deletedAt` timestamp
- Trash view to browse deleted items
- Restore or permanent delete

#### 9. Search with Full-Text Indexing
**What**: PostgreSQL full-text search with tsvector
**Why**: Fast, relevant search across all content

**Technical Details**:
- `tsvector` column with GIN index
- Automatic update on content changes
- Ranked search results

---

### 🚀 Low Priority (Advanced Features)

#### 10. Real-Time Collaboration (Yjs)
**What**: Multiple users editing simultaneously
**Why**: Team editing for documentation

**Complexity**: High (requires WebSocket infrastructure)
**Consider**: Start with single-user, add later if needed

#### 11. Public Sharing Links
**What**: Generate shareable links with custom permissions
**Why**: Share specific pages publicly without authentication

#### 12. Backlinks System
**What**: Track page references to show "linked from"
**Why**: Knowledge graph navigation

---

## Integration Routes

I've identified **three routes** with increasing complexity and capabilities:

---

## Route 1: Minimal Integration (Quick Wins)
**Timeline**: 1-2 weeks
**Complexity**: Low
**Impact**: Moderate

### What to Implement
1. ✅ Add hierarchical page model (parent-child relationships)
2. ✅ Implement fractional indexing for page ordering
3. ✅ Enhanced attachment metadata (size, mime type, associations)
4. ✅ Add 3-5 key Tiptap extensions (Math, Callouts, Details)
5. ✅ Page metadata (icon, cover, position)

### What to Skip
- Real-time collaboration
- Storage driver abstraction
- Page history
- Complex search

### Database Changes
```prisma
model Page {
  id            String   @id @default(cuid())
  slug          String   @unique
  title         String
  content       Json     // Existing TipTap JSON

  // NEW FIELDS
  icon          String?
  coverPhoto    String?
  position      String   // Fractional index
  parentId      String?
  parent        Page?    @relation("PageHierarchy", fields: [parentId], references: [id])
  children      Page[]   @relation("PageHierarchy")

  // Existing fields...
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  @@index([parentId])
  @@index([position])
}

model Attachment {
  id            String   @id @default(cuid())
  fileName      String
  filePath      String
  fileSize      Int      // NEW
  mimeType      String   // NEW

  // Multiple page associations
  pages         Page[]   @relation("PageAttachments")

  uploadedAt    DateTime @default(now())
}
```

### Implementation Steps
1. Create database migration with new schema
2. Install `fractional-indexing-jittered` package
3. Create page hierarchy utilities (get children, breadcrumbs)
4. Update page creation/editing forms with parent selector
5. Add icon and cover photo pickers
6. Integrate new Tiptap extensions
7. Update page display with hierarchy navigation

### Pros
✅ Quick to implement
✅ Immediate value (better organization)
✅ Low risk (minimal changes)
✅ Works with existing Next.js architecture

### Cons
❌ No real-time features
❌ Still using Prisma (less flexible than Kysely)
❌ No advanced search
❌ Limited versioning

---

## Route 2: Comprehensive Integration (Recommended)
**Timeline**: 4-6 weeks
**Complexity**: Medium
**Impact**: High

### What to Implement
1. ✅ Everything from Route 1
2. ✅ Storage driver abstraction layer
3. ✅ Page history and versioning
4. ✅ Soft deletes with trash system
5. ✅ Full-text search with tsvector
6. ✅ All relevant Tiptap extensions
7. ✅ Attachment cleanup job queue
8. ✅ Public sharing links (optional)

### What to Skip
- Real-time collaboration (add later if needed)
- Migration to Kysely (stay with Prisma)
- Complex workspace/space hierarchy (keep it simple)

### Database Changes
```prisma
model Page {
  id            String   @id @default(cuid())
  slug          String   @unique
  title         String
  content       Json
  textContent   String   // Plain text for search

  // Hierarchy
  icon          String?
  coverPhoto    String?
  position      String
  parentId      String?
  parent        Page?    @relation("PageHierarchy", fields: [parentId], references: [id])
  children      Page[]   @relation("PageHierarchy")

  // Soft delete
  deletedAt     DateTime?
  deletedBy     String?

  // Metadata
  isLocked      Boolean  @default(false)
  viewCount     Int      @default(0)

  // Associations
  attachments   Attachment[] @relation("PageAttachments")
  history       PageHistory[]
  shares        Share[]

  // Timestamps
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  createdBy     String?

  @@index([parentId])
  @@index([position])
  @@index([deletedAt])
  @@fulltext([title, textContent])
}

model PageHistory {
  id            String   @id @default(cuid())
  pageId        String
  page          Page     @relation(fields: [pageId], references: [id], onDelete: Cascade)

  content       Json
  title         String

  createdAt     DateTime @default(now())
  createdBy     String?

  @@index([pageId, createdAt])
}

model Attachment {
  id            String   @id @default(cuid())
  fileName      String
  filePath      String
  fileSize      Int
  mimeType      String
  fileExt       String

  // Associations
  pages         Page[]   @relation("PageAttachments")

  // OCR/Text extraction for PDFs
  textContent   String?

  // Metadata
  uploadedAt    DateTime @default(now())
  uploadedBy    String?

  @@index([mimeType])
}

model Share {
  id            String   @id @default(cuid())
  pageId        String
  page          Page     @relation(fields: [pageId], references: [id], onDelete: Cascade)

  token         String   @unique
  expiresAt     DateTime?

  createdAt     DateTime @default(now())
  createdBy     String?

  @@index([token])
}
```

### Storage Driver Architecture
```typescript
// src/lib/storage/driver.ts
export interface StorageDriver {
  upload(path: string, content: Buffer): Promise<void>;
  read(path: string): Promise<Buffer>;
  delete(path: string): Promise<void>;
  exists(path: string): Promise<boolean>;
  getUrl(path: string): string;
  getSignedUrl(path: string, expiresIn: number): Promise<string>;
}

// src/lib/storage/drivers/vercel-blob.ts
export class VercelBlobDriver implements StorageDriver {
  // Implementation
}

// src/lib/storage/drivers/s3.ts
export class S3Driver implements StorageDriver {
  // Implementation
}

// src/lib/storage/index.ts
export function getStorageDriver(): StorageDriver {
  const driver = process.env.STORAGE_DRIVER || 'vercel-blob';
  switch (driver) {
    case 'vercel-blob': return new VercelBlobDriver();
    case 's3': return new S3Driver();
    default: throw new Error(`Unknown storage driver: ${driver}`);
  }
}
```

### Full-Text Search Implementation
```typescript
// Prisma migration for tsvector
// Add to Page model:
// textVector    Unsupported("tsvector")?

// Migration SQL:
ALTER TABLE "Page" ADD COLUMN "textVector" tsvector;
CREATE INDEX "Page_textVector_idx" ON "Page" USING GIN ("textVector");

CREATE OR REPLACE FUNCTION page_tsvector_trigger() RETURNS trigger AS $$
BEGIN
  NEW."textVector" := to_tsvector('english', COALESCE(NEW.title, '') || ' ' || COALESCE(NEW."textContent", ''));
  RETURN NEW;
END
$$ LANGUAGE plpgsql;

CREATE TRIGGER page_tsvector_update
  BEFORE INSERT OR UPDATE ON "Page"
  FOR EACH ROW EXECUTE FUNCTION page_tsvector_trigger();
```

### Implementation Steps
1. **Week 1**: Database schema migration + page hierarchy
2. **Week 2**: Fractional indexing + page CRUD operations
3. **Week 3**: Storage driver abstraction + attachment system
4. **Week 4**: Tiptap extensions + editor enhancements
5. **Week 5**: Page history + soft deletes + trash UI
6. **Week 6**: Full-text search + public sharing + testing

### Pros
✅ Production-ready document management
✅ Comprehensive feature set
✅ Maintains Prisma (familiar)
✅ Flexible storage backends
✅ Full-text search capabilities
✅ Versioning and recovery

### Cons
❌ More development time
❌ More complex testing
❌ Still no real-time collaboration

---

## Route 3: Full Docmost Architecture (Complex)
**Timeline**: 8-12 weeks
**Complexity**: High
**Impact**: Very High

### What to Implement
1. ✅ Everything from Route 2
2. ✅ Real-time collaboration with Yjs + Hocuspocus
3. ✅ Migration from Prisma to Kysely (optional but recommended for flexibility)
4. ✅ WebSocket infrastructure for real-time sync
5. ✅ Workspace → Spaces → Pages hierarchy
6. ✅ Permission system with CASL
7. ✅ Background job queue with BullMQ
8. ✅ Backlinks system

### Architecture Changes
- Add WebSocket server (Hocuspocus) alongside Next.js
- Store both ProseMirror JSON and Yjs binary documents
- Real-time tree synchronization
- IndexedDB for offline support

### Database Changes
```prisma
model Workspace {
  id            String   @id @default(cuid())
  name          String
  slug          String   @unique

  spaces        Space[]
  users         User[]
}

model Space {
  id            String   @id @default(cuid())
  workspaceId   String
  workspace     Workspace @relation(fields: [workspaceId], references: [id])

  name          String
  slug          String
  visibility    String   @default("private") // private, public

  pages         Page[]
  members       SpaceMember[]
}

model Page {
  id            String   @id @default(cuid())
  spaceId       String
  space         Space    @relation(fields: [spaceId], references: [id], onDelete: Cascade)

  // Content (dual storage for collaboration)
  content       Json     // ProseMirror JSON
  ydoc          Bytes?   // Yjs CRDT binary
  textContent   String   // Plain text for search

  // ... all fields from Route 2 ...

  backlinksFrom Backlink[] @relation("SourcePage")
  backlinksTo   Backlink[] @relation("TargetPage")
}

model Backlink {
  id            String   @id @default(cuid())
  sourcePageId  String
  sourcePage    Page     @relation("SourcePage", fields: [sourcePageId], references: [id], onDelete: Cascade)
  targetPageId  String
  targetPage    Page     @relation("TargetPage", fields: [targetPageId], references: [id], onDelete: Cascade)

  createdAt     DateTime @default(now())

  @@unique([sourcePageId, targetPageId])
}
```

### Collaboration Architecture
```typescript
// server/collab.ts
import { Server } from '@hocuspocus/server';
import { Database } from '@hocuspocus/extension-database';
import { Redis } from '@hocuspocus/extension-redis';

const server = Server.configure({
  extensions: [
    new Database({
      fetch: async ({ documentName }) => {
        const page = await prisma.page.findUnique({
          where: { id: documentName }
        });
        return page?.ydoc || null;
      },
      store: async ({ documentName, state }) => {
        await prisma.page.update({
          where: { id: documentName },
          data: { ydoc: Buffer.from(state) }
        });
      }
    }),
    new Redis({
      host: process.env.REDIS_HOST,
      port: process.env.REDIS_PORT
    })
  ]
});

server.listen(1234);
```

### Implementation Steps
1. **Weeks 1-2**: Complete Route 2 implementation
2. **Week 3**: Set up Hocuspocus server + WebSocket infrastructure
3. **Week 4**: Integrate Yjs collaboration extension in editor
4. **Week 5**: Workspace/Space hierarchy + permissions
5. **Week 6**: Real-time tree synchronization
6. **Week 7**: Background job queue for indexing/cleanup
7. **Week 8**: Backlinks system
8. **Weeks 9-10**: Testing, optimization, bug fixes
9. **Weeks 11-12**: Migration to Kysely (optional)

### Pros
✅ Feature parity with Docmost
✅ Real-time collaboration
✅ Multi-workspace support
✅ Advanced permissions
✅ Production-grade scalability
✅ Backlinks and knowledge graph

### Cons
❌ Longest development time
❌ Highest complexity
❌ Requires WebSocket infrastructure
❌ Potential hosting cost increase
❌ More complex deployment

---

## Detailed Implementation Plans

### Plan A: Hierarchical Pages (All Routes)

#### 1. Database Migration
```prisma
// Add to schema.prisma
model Page {
  // ... existing fields ...

  position      String   @default("a0")
  parentId      String?
  parent        Page?    @relation("PageHierarchy", fields: [parentId], references: [id])
  children      Page[]   @relation("PageHierarchy")

  @@index([parentId])
  @@index([position])
}
```

#### 2. Install Dependencies
```bash
pnpm add fractional-indexing-jittered
```

#### 3. Create Utilities
```typescript
// src/lib/fractional-index.ts
import { generateJitteredKeyBetween } from 'fractional-indexing-jittered';

export function getPositionBefore(position: string): string {
  return generateJitteredKeyBetween(null, position);
}

export function getPositionAfter(position: string): string {
  return generateJitteredKeyBetween(position, null);
}

export function getPositionBetween(before: string, after: string): string {
  return generateJitteredKeyBetween(before, after);
}

// src/lib/page-tree.ts
export async function getPageBreadcrumbs(pageId: string) {
  const breadcrumbs = [];
  let currentPage = await prisma.page.findUnique({ where: { id: pageId } });

  while (currentPage) {
    breadcrumbs.unshift({
      id: currentPage.id,
      title: currentPage.title,
      slug: currentPage.slug
    });

    if (currentPage.parentId) {
      currentPage = await prisma.page.findUnique({
        where: { id: currentPage.parentId }
      });
    } else {
      break;
    }
  }

  return breadcrumbs;
}

export async function getPageTree(parentId: string | null = null) {
  const pages = await prisma.page.findMany({
    where: { parentId },
    orderBy: { position: 'asc' }
  });

  return pages;
}
```

#### 4. Create Server Actions
```typescript
// src/app/actions/page-actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { prisma } from '@/lib/db';
import { getPositionAfter } from '@/lib/fractional-index';

export async function createPage(data: {
  title: string;
  parentId?: string;
  content?: any;
}) {
  // Get the last sibling to determine position
  const lastSibling = await prisma.page.findFirst({
    where: { parentId: data.parentId || null },
    orderBy: { position: 'desc' }
  });

  const position = lastSibling
    ? getPositionAfter(lastSibling.position)
    : 'a0';

  const page = await prisma.page.create({
    data: {
      title: data.title,
      slug: generateSlug(data.title),
      parentId: data.parentId,
      position,
      content: data.content || {}
    }
  });

  revalidatePath('/pages');
  return page;
}

export async function movePage(
  pageId: string,
  newParentId: string | null,
  afterPageId: string | null
) {
  let newPosition: string;

  if (afterPageId) {
    // Get the page after which we want to insert
    const afterPage = await prisma.page.findUnique({
      where: { id: afterPageId }
    });

    // Get the next sibling
    const nextSibling = await prisma.page.findFirst({
      where: {
        parentId: newParentId,
        position: { gt: afterPage!.position }
      },
      orderBy: { position: 'asc' }
    });

    newPosition = getPositionBetween(
      afterPage!.position,
      nextSibling?.position || null
    );
  } else {
    // Insert at the beginning
    const firstSibling = await prisma.page.findFirst({
      where: { parentId: newParentId },
      orderBy: { position: 'asc' }
    });

    newPosition = getPositionBefore(firstSibling?.position || 'a0');
  }

  await prisma.page.update({
    where: { id: pageId },
    data: {
      parentId: newParentId,
      position: newPosition
    }
  });

  revalidatePath('/pages');
}
```

#### 5. Create UI Components
```typescript
// src/components/page-tree.tsx
'use client';

import { useState } from 'react';
import { ChevronRight, ChevronDown, FileText } from 'lucide-react';
import Link from 'next/link';

interface PageNode {
  id: string;
  title: string;
  slug: string;
  children: PageNode[];
}

export function PageTree({ pages }: { pages: PageNode[] }) {
  return (
    <ul className="space-y-1">
      {pages.map(page => (
        <PageTreeNode key={page.id} page={page} />
      ))}
    </ul>
  );
}

function PageTreeNode({ page }: { page: PageNode }) {
  const [isOpen, setIsOpen] = useState(false);
  const hasChildren = page.children.length > 0;

  return (
    <li>
      <div className="flex items-center gap-2 hover:bg-gray-100 rounded px-2 py-1">
        {hasChildren ? (
          <button onClick={() => setIsOpen(!isOpen)} className="p-0.5">
            {isOpen ? <ChevronDown size={16} /> : <ChevronRight size={16} />}
          </button>
        ) : (
          <FileText size={16} className="ml-6" />
        )}

        <Link href={`/pages/${page.slug}`} className="flex-1">
          {page.title}
        </Link>
      </div>

      {hasChildren && isOpen && (
        <ul className="ml-4 mt-1">
          {page.children.map(child => (
            <PageTreeNode key={child.id} page={child} />
          ))}
        </ul>
      )}
    </li>
  );
}
```

---

### Plan B: Enhanced Tiptap Extensions (Routes 1 & 2)

#### 1. Install Extension Dependencies
```bash
# Math extensions
pnpm add katex @types/katex

# Markdown support
pnpm add marked @joplin/turndown @joplin/turndown-plugin-gfm

# Diagram support (optional)
pnpm add @excalidraw/excalidraw mermaid
```

#### 2. Port Extensions from Docmost
Copy these files from `docmost-main/packages/editor-ext/src/lib/`:
- `math/` → Math inline and block
- `callout/` → Styled callouts
- `details/` → Collapsible sections
- `search-and-replace/` → Find/replace
- `custom-code-block.ts` → Enhanced code blocks

#### 3. Create Extension Package (Optional)
```typescript
// src/lib/editor/extensions/index.ts
export { TiptapMathInline } from './math/math-inline';
export { TiptapMathBlock } from './math/math-block';
export { Callout } from './callout/callout';
export { Details, DetailsContent, DetailsSummary } from './details';
export { SearchAndReplace } from './search-and-replace';
```

#### 4. Update Editor Configuration
```typescript
// src/components/editor/extensions.ts
import StarterKit from '@tiptap/starter-kit';
import { TiptapMathInline, TiptapMathBlock } from '@/lib/editor/extensions';
import { Callout } from '@/lib/editor/extensions';
import { Details, DetailsContent, DetailsSummary } from '@/lib/editor/extensions';

export const extensions = [
  StarterKit.configure({
    // Your existing config
  }),

  // Math support
  TiptapMathInline,
  TiptapMathBlock,

  // Callouts
  Callout,

  // Collapsible sections
  Details,
  DetailsContent,
  DetailsSummary,

  // ... your existing extensions
];
```

#### 5. Add Extension UI Components
```typescript
// src/components/editor/math-menu.tsx
import { Button } from '@/components/ui/button';
import { Calculator } from 'lucide-react';

export function MathMenu({ editor }: { editor: Editor }) {
  const addInlineMath = () => {
    editor.chain().focus().insertContent({ type: 'mathInline' }).run();
  };

  const addBlockMath = () => {
    editor.chain().focus().insertContent({ type: 'mathBlock' }).run();
  };

  return (
    <>
      <Button onClick={addInlineMath} size="sm" variant="ghost">
        <Calculator size={16} />
        <span>Inline Math</span>
      </Button>
      <Button onClick={addBlockMath} size="sm" variant="ghost">
        <Calculator size={16} />
        <span>Block Math</span>
      </Button>
    </>
  );
}

// src/components/editor/callout-menu.tsx
export function CalloutMenu({ editor }: { editor: Editor }) {
  const addCallout = (type: 'info' | 'warning' | 'success' | 'error') => {
    editor.chain().focus().insertContent({
      type: 'callout',
      attrs: { type }
    }).run();
  };

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button size="sm" variant="ghost">
          <AlertCircle size={16} />
          <span>Callout</span>
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent>
        <DropdownMenuItem onClick={() => addCallout('info')}>
          Info
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => addCallout('warning')}>
          Warning
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => addCallout('success')}>
          Success
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => addCallout('error')}>
          Error
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

---

### Plan C: Storage Driver Abstraction (Route 2 & 3)

#### 1. Define Storage Interface
```typescript
// src/lib/storage/types.ts
export interface UploadOptions {
  contentType?: string;
  metadata?: Record<string, string>;
}

export interface StorageDriver {
  upload(path: string, content: Buffer | Blob, options?: UploadOptions): Promise<void>;
  uploadStream(path: string, stream: ReadableStream, options?: UploadOptions): Promise<void>;
  read(path: string): Promise<Buffer>;
  readStream(path: string): Promise<ReadableStream>;
  delete(path: string): Promise<void>;
  exists(path: string): Promise<boolean>;
  copy(from: string, to: string): Promise<void>;
  getUrl(path: string): string;
  getSignedUrl(path: string, expiresIn: number): Promise<string>;
}
```

#### 2. Implement Drivers
```typescript
// src/lib/storage/drivers/vercel-blob.ts
import { put, del, head, copy } from '@vercel/blob';
import { StorageDriver, UploadOptions } from '../types';

export class VercelBlobDriver implements StorageDriver {
  private baseUrl: string;

  constructor() {
    this.baseUrl = process.env.BLOB_READ_WRITE_TOKEN || '';
  }

  async upload(path: string, content: Buffer | Blob, options?: UploadOptions): Promise<void> {
    await put(path, content, {
      access: 'public',
      contentType: options?.contentType,
      addRandomSuffix: false
    });
  }

  async read(path: string): Promise<Buffer> {
    const response = await fetch(this.getUrl(path));
    const arrayBuffer = await response.arrayBuffer();
    return Buffer.from(arrayBuffer);
  }

  async delete(path: string): Promise<void> {
    await del(this.getUrl(path));
  }

  async exists(path: string): Promise<boolean> {
    try {
      await head(this.getUrl(path));
      return true;
    } catch {
      return false;
    }
  }

  async copy(from: string, to: string): Promise<void> {
    await copy(this.getUrl(from), to, { access: 'public' });
  }

  getUrl(path: string): string {
    return `${this.baseUrl}/${path}`;
  }

  async getSignedUrl(path: string, expiresIn: number): Promise<string> {
    // Vercel Blob URLs are already public or token-based
    return this.getUrl(path);
  }

  // Implement other methods...
}

// src/lib/storage/drivers/s3.ts
import { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand, HeadObjectCommand, CopyObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl as getS3SignedUrl } from '@aws-sdk/s3-request-presigner';
import { StorageDriver, UploadOptions } from '../types';

export class S3Driver implements StorageDriver {
  private client: S3Client;
  private bucket: string;
  private region: string;

  constructor() {
    this.bucket = process.env.AWS_S3_BUCKET || '';
    this.region = process.env.AWS_REGION || 'us-east-1';

    this.client = new S3Client({
      region: this.region,
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID || '',
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY || ''
      }
    });
  }

  async upload(path: string, content: Buffer | Blob, options?: UploadOptions): Promise<void> {
    const buffer = content instanceof Buffer ? content : Buffer.from(await content.arrayBuffer());

    await this.client.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: path,
      Body: buffer,
      ContentType: options?.contentType,
      Metadata: options?.metadata
    }));
  }

  async read(path: string): Promise<Buffer> {
    const response = await this.client.send(new GetObjectCommand({
      Bucket: this.bucket,
      Key: path
    }));

    const chunks: Uint8Array[] = [];
    for await (const chunk of response.Body as any) {
      chunks.push(chunk);
    }
    return Buffer.concat(chunks);
  }

  async delete(path: string): Promise<void> {
    await this.client.send(new DeleteObjectCommand({
      Bucket: this.bucket,
      Key: path
    }));
  }

  async exists(path: string): Promise<boolean> {
    try {
      await this.client.send(new HeadObjectCommand({
        Bucket: this.bucket,
        Key: path
      }));
      return true;
    } catch {
      return false;
    }
  }

  async copy(from: string, to: string): Promise<void> {
    await this.client.send(new CopyObjectCommand({
      Bucket: this.bucket,
      CopySource: `${this.bucket}/${from}`,
      Key: to
    }));
  }

  getUrl(path: string): string {
    return `https://${this.bucket}.s3.${this.region}.amazonaws.com/${path}`;
  }

  async getSignedUrl(path: string, expiresIn: number): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: path
    });

    return await getS3SignedUrl(this.client, command, { expiresIn });
  }

  // Implement other methods...
}
```

#### 3. Create Storage Factory
```typescript
// src/lib/storage/index.ts
import { StorageDriver } from './types';
import { VercelBlobDriver } from './drivers/vercel-blob';
import { S3Driver } from './drivers/s3';
import { R2Driver } from './drivers/r2';

export function createStorageDriver(): StorageDriver {
  const driverType = process.env.STORAGE_DRIVER || 'vercel-blob';

  switch (driverType) {
    case 'vercel-blob':
      return new VercelBlobDriver();
    case 's3':
      return new S3Driver();
    case 'r2':
      return new R2Driver();
    default:
      throw new Error(`Unknown storage driver: ${driverType}`);
  }
}

// Singleton instance
let storageInstance: StorageDriver | null = null;

export function getStorage(): StorageDriver {
  if (!storageInstance) {
    storageInstance = createStorageDriver();
  }
  return storageInstance;
}
```

#### 4. Update Upload Actions
```typescript
// src/app/api/upload/route.ts
import { getStorage } from '@/lib/storage';
import { prisma } from '@/lib/db';

export async function POST(request: Request) {
  const formData = await request.formData();
  const file = formData.get('file') as File;
  const pageId = formData.get('pageId') as string;

  if (!file) {
    return Response.json({ error: 'No file provided' }, { status: 400 });
  }

  const storage = getStorage();
  const attachmentId = generateId();
  const filePath = `pages/${pageId}/attachments/${attachmentId}/${file.name}`;

  // Upload to storage
  const buffer = Buffer.from(await file.arrayBuffer());
  await storage.upload(filePath, buffer, {
    contentType: file.type
  });

  // Save metadata to database
  const attachment = await prisma.attachment.create({
    data: {
      id: attachmentId,
      fileName: file.name,
      filePath,
      fileSize: file.size,
      mimeType: file.type,
      fileExt: getFileExtension(file.name),
      pageId
    }
  });

  return Response.json({
    id: attachment.id,
    fileName: attachment.fileName,
    fileSize: attachment.fileSize,
    url: storage.getUrl(filePath)
  });
}
```

---

### Plan D: Page History (Route 2 & 3)

#### 1. Database Schema
```prisma
model PageHistory {
  id            String   @id @default(cuid())
  pageId        String
  page          Page     @relation(fields: [pageId], references: [id], onDelete: Cascade)

  content       Json
  title         String
  textContent   String?

  createdAt     DateTime @default(now())
  createdBy     String?
  createdByUser User?    @relation(fields: [createdBy], references: [id])

  @@index([pageId, createdAt])
}
```

#### 2. Snapshot Creation
```typescript
// src/lib/page-history.ts
import { prisma } from './db';

export async function createSnapshot(pageId: string, userId?: string) {
  const page = await prisma.page.findUnique({
    where: { id: pageId }
  });

  if (!page) throw new Error('Page not found');

  await prisma.pageHistory.create({
    data: {
      pageId,
      content: page.content,
      title: page.title,
      textContent: page.textContent,
      createdBy: userId
    }
  });
}

export async function getPageHistory(pageId: string, limit: number = 50) {
  return await prisma.pageHistory.findMany({
    where: { pageId },
    orderBy: { createdAt: 'desc' },
    take: limit,
    include: {
      createdByUser: {
        select: { name: true, email: true }
      }
    }
  });
}

export async function restoreFromHistory(historyId: string) {
  const history = await prisma.pageHistory.findUnique({
    where: { id: historyId }
  });

  if (!history) throw new Error('History not found');

  // Create snapshot of current state before restoring
  await createSnapshot(history.pageId);

  // Restore page content
  await prisma.page.update({
    where: { id: history.pageId },
    data: {
      content: history.content,
      title: history.title,
      textContent: history.textContent
    }
  });
}
```

#### 3. Auto-Snapshot on Update
```typescript
// src/app/actions/page-actions.ts
export async function updatePage(pageId: string, data: {
  title?: string;
  content?: any;
}) {
  // Create snapshot before updating
  await createSnapshot(pageId, getCurrentUserId());

  const updated = await prisma.page.update({
    where: { id: pageId },
    data: {
      ...data,
      textContent: extractTextFromContent(data.content),
      updatedAt: new Date()
    }
  });

  revalidatePath(`/pages/${updated.slug}`);
  return updated;
}
```

#### 4. History UI
```typescript
// src/components/page-history-dialog.tsx
'use client';

import { useState } from 'react';
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';
import { formatDistanceToNow } from 'date-fns';

export function PageHistoryDialog({ pageId }: { pageId: string }) {
  const [history, setHistory] = useState([]);
  const [selectedVersion, setSelectedVersion] = useState(null);

  // Fetch history
  // ...

  return (
    <Dialog>
      <DialogContent className="max-w-4xl max-h-[80vh] overflow-y-auto">
        <DialogHeader>
          <DialogTitle>Page History</DialogTitle>
        </DialogHeader>

        <div className="grid grid-cols-3 gap-4">
          <div className="col-span-1 border-r pr-4">
            <h3 className="font-semibold mb-2">Versions</h3>
            <ul className="space-y-2">
              {history.map(version => (
                <li key={version.id}>
                  <button
                    onClick={() => setSelectedVersion(version)}
                    className="text-left w-full p-2 hover:bg-gray-100 rounded"
                  >
                    <div className="font-medium">{version.title}</div>
                    <div className="text-sm text-gray-500">
                      {formatDistanceToNow(new Date(version.createdAt))} ago
                    </div>
                    <div className="text-xs text-gray-400">
                      by {version.createdByUser?.name || 'Unknown'}
                    </div>
                  </button>
                </li>
              ))}
            </ul>
          </div>

          <div className="col-span-2">
            {selectedVersion ? (
              <>
                <div className="flex justify-between items-center mb-4">
                  <h3 className="font-semibold">Preview</h3>
                  <Button
                    onClick={() => restoreVersion(selectedVersion.id)}
                    variant="default"
                  >
                    Restore This Version
                  </Button>
                </div>

                <div className="prose max-w-none">
                  {/* Render content preview */}
                </div>
              </>
            ) : (
              <div className="text-center text-gray-500 py-8">
                Select a version to preview
              </div>
            )}
          </div>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

---

### Plan E: Full-Text Search (Route 2 & 3)

#### 1. Database Setup
```sql
-- Migration: Add tsvector column
ALTER TABLE "Page" ADD COLUMN "textVector" tsvector;

-- Create GIN index for fast full-text search
CREATE INDEX "Page_textVector_idx" ON "Page" USING GIN ("textVector");

-- Create trigger function to auto-update tsvector
CREATE OR REPLACE FUNCTION page_tsvector_trigger() RETURNS trigger AS $$
BEGIN
  NEW."textVector" :=
    setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW."textContent", '')), 'B');
  RETURN NEW;
END
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER page_tsvector_update
  BEFORE INSERT OR UPDATE ON "Page"
  FOR EACH ROW
  EXECUTE FUNCTION page_tsvector_trigger();

-- Backfill existing records
UPDATE "Page" SET "textVector" =
  setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
  setweight(to_tsvector('english', COALESCE("textContent", '')), 'B');
```

#### 2. Search Function
```typescript
// src/lib/search.ts
import { prisma } from './db';

export interface SearchResult {
  id: string;
  title: string;
  slug: string;
  excerpt: string;
  rank: number;
  headline: string;
}

export async function searchPages(query: string, limit: number = 20): Promise<SearchResult[]> {
  // Use raw SQL for full-text search with ranking
  const results = await prisma.$queryRaw<SearchResult[]>`
    SELECT
      id,
      title,
      slug,
      ts_headline('english', "textContent", plainto_tsquery('english', ${query}),
        'MaxWords=50, MinWords=25, MaxFragments=1') as excerpt,
      ts_rank("textVector", plainto_tsquery('english', ${query})) as rank,
      ts_headline('english', title || ' ' || COALESCE("textContent", ''),
        plainto_tsquery('english', ${query}), 'MaxWords=10') as headline
    FROM "Page"
    WHERE
      "textVector" @@ plainto_tsquery('english', ${query})
      AND "deletedAt" IS NULL
    ORDER BY rank DESC
    LIMIT ${limit}
  `;

  return results;
}

export async function searchPagesWithFilter(
  query: string,
  filters: {
    parentId?: string;
    createdBy?: string;
    createdAfter?: Date;
  } = {},
  limit: number = 20
): Promise<SearchResult[]> {
  const whereConditions = [
    `"textVector" @@ plainto_tsquery('english', '${query}')`,
    `"deletedAt" IS NULL`
  ];

  if (filters.parentId) {
    whereConditions.push(`"parentId" = '${filters.parentId}'`);
  }

  if (filters.createdBy) {
    whereConditions.push(`"createdBy" = '${filters.createdBy}'`);
  }

  if (filters.createdAfter) {
    whereConditions.push(`"createdAt" >= '${filters.createdAfter.toISOString()}'`);
  }

  const whereClause = whereConditions.join(' AND ');

  const results = await prisma.$queryRaw<SearchResult[]>`
    SELECT
      id,
      title,
      slug,
      ts_headline('english', "textContent", plainto_tsquery('english', ${query})) as excerpt,
      ts_rank("textVector", plainto_tsquery('english', ${query})) as rank
    FROM "Page"
    WHERE ${whereClause}
    ORDER BY rank DESC
    LIMIT ${limit}
  `;

  return results;
}
```

#### 3. Search API Route
```typescript
// src/app/api/search/route.ts
import { NextRequest } from 'next/server';
import { searchPages } from '@/lib/search';

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const query = searchParams.get('q');
  const limit = parseInt(searchParams.get('limit') || '20');

  if (!query || query.trim().length === 0) {
    return Response.json({ results: [] });
  }

  try {
    const results = await searchPages(query, limit);
    return Response.json({ results, count: results.length });
  } catch (error) {
    console.error('Search error:', error);
    return Response.json(
      { error: 'Search failed' },
      { status: 500 }
    );
  }
}
```

#### 4. Search UI Component
```typescript
// src/components/search-dialog.tsx
'use client';

import { useState, useCallback } from 'react';
import { useDebounce } from '@/hooks/use-debounce';
import { CommandDialog, CommandInput, CommandList, CommandEmpty, CommandGroup, CommandItem } from '@/components/ui/command';
import { Search, FileText } from 'lucide-react';
import { useRouter } from 'next/navigation';

export function SearchDialog() {
  const [open, setOpen] = useState(false);
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const router = useRouter();

  const debouncedQuery = useDebounce(query, 300);

  // Listen for Cmd+K
  useEffect(() => {
    const down = (e: KeyboardEvent) => {
      if (e.key === 'k' && (e.metaKey || e.ctrlKey)) {
        e.preventDefault();
        setOpen(open => !open);
      }
    };

    document.addEventListener('keydown', down);
    return () => document.removeEventListener('keydown', down);
  }, []);

  // Perform search
  useEffect(() => {
    if (debouncedQuery.length < 2) {
      setResults([]);
      return;
    }

    setLoading(true);
    fetch(`/api/search?q=${encodeURIComponent(debouncedQuery)}`)
      .then(res => res.json())
      .then(data => {
        setResults(data.results);
        setLoading(false);
      })
      .catch(() => {
        setLoading(false);
      });
  }, [debouncedQuery]);

  const handleSelect = useCallback((slug: string) => {
    setOpen(false);
    router.push(`/pages/${slug}`);
  }, [router]);

  return (
    <>
      <button
        onClick={() => setOpen(true)}
        className="flex items-center gap-2 px-3 py-1.5 text-sm text-gray-600 border rounded-md hover:bg-gray-50"
      >
        <Search size={16} />
        <span>Search pages...</span>
        <kbd className="ml-auto text-xs">⌘K</kbd>
      </button>

      <CommandDialog open={open} onOpenChange={setOpen}>
        <CommandInput
          placeholder="Search pages..."
          value={query}
          onValueChange={setQuery}
        />
        <CommandList>
          {loading && (
            <div className="p-4 text-sm text-center text-gray-500">
              Searching...
            </div>
          )}

          {!loading && results.length === 0 && query.length >= 2 && (
            <CommandEmpty>No results found.</CommandEmpty>
          )}

          {!loading && results.length > 0 && (
            <CommandGroup heading="Pages">
              {results.map((result: any) => (
                <CommandItem
                  key={result.id}
                  value={result.id}
                  onSelect={() => handleSelect(result.slug)}
                >
                  <FileText className="mr-2 h-4 w-4" />
                  <div className="flex-1">
                    <div
                      className="font-medium"
                      dangerouslySetInnerHTML={{ __html: result.headline }}
                    />
                    <div
                      className="text-xs text-gray-500 line-clamp-1"
                      dangerouslySetInnerHTML={{ __html: result.excerpt }}
                    />
                  </div>
                </CommandItem>
              ))}
            </CommandGroup>
          )}
        </CommandList>
      </CommandDialog>
    </>
  );
}
```

---

## Technical Specifications

### Performance Considerations

#### 1. Fractional Indexing
- **Advantage**: O(1) reordering without updating siblings
- **Consideration**: Position strings grow over time
- **Solution**: Periodic rebalancing (optional)

#### 2. Hierarchical Queries
- **Challenge**: Deep nesting can cause slow recursive queries
- **Solution**: Use PostgreSQL recursive CTEs
- **Optimization**: Limit recursion depth, cache breadcrumbs

#### 3. Full-Text Search
- **Index Type**: GIN index on tsvector column
- **Update Strategy**: Trigger-based auto-update
- **Query Performance**: <50ms for most queries
- **Scaling**: Consider PostgreSQL FTS limits at 100k+ pages

#### 4. File Storage
- **CDN**: Use CloudFront (S3) or Vercel Edge for file delivery
- **Optimization**: Compress images, lazy load media
- **Cleanup**: Background job for orphaned files

### Security Considerations

#### 1. File Uploads
```typescript
// Validate file types and sizes
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'application/pdf'];
const MAX_SIZE = 10 * 1024 * 1024; // 10MB

if (!ALLOWED_TYPES.includes(file.type)) {
  throw new Error('Invalid file type');
}

if (file.size > MAX_SIZE) {
  throw new Error('File too large');
}

// Sanitize filenames
function sanitizeFilename(filename: string): string {
  return filename
    .replace(/[^a-zA-Z0-9.-]/g, '_')
    .replace(/\.+/g, '.')
    .toLowerCase();
}
```

#### 2. Access Control
```typescript
// Check page access before serving
async function canAccessPage(userId: string, pageId: string): Promise<boolean> {
  const page = await prisma.page.findUnique({
    where: { id: pageId },
    include: { space: true }
  });

  if (!page) return false;

  // Check if user is in workspace
  // Check if page is public
  // Check if user has space membership

  return true;
}
```

#### 3. SQL Injection Prevention
- Always use parameterized queries
- Prisma automatically escapes values
- For raw SQL, use `prisma.$queryRaw` with template literals

### Scalability Considerations

#### 1. Database
- **Connection Pooling**: Use PgBouncer for connection management
- **Indexes**: Ensure proper indexes on `parentId`, `position`, `deletedAt`
- **Partitioning**: Consider table partitioning for large page counts (1M+)

#### 2. File Storage
- **CDN**: Essential for global performance
- **Pre-signed URLs**: Offload auth to storage provider
- **Async Processing**: Queue thumbnail generation, OCR

#### 3. Caching
```typescript
// Cache page tree in Redis
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_URL,
  token: process.env.UPSTASH_REDIS_TOKEN
});

export async function getCachedPageTree(spaceId: string) {
  const cached = await redis.get(`page-tree:${spaceId}`);
  if (cached) return cached;

  const tree = await buildPageTree(spaceId);
  await redis.set(`page-tree:${spaceId}`, tree, { ex: 300 }); // 5 min

  return tree;
}
```

---

## Migration Strategy

### Phase 1: Preparation (Week 1)
1. **Backup Database**: Full PostgreSQL dump
2. **Feature Flag**: Implement feature flag for new features
3. **Parallel Development**: New features behind flag
4. **Testing Environment**: Set up staging environment

### Phase 2: Schema Migration (Week 2)
1. **Add New Columns**: Add nullable columns first
2. **Backfill Data**: Populate `position`, `textContent` for existing pages
3. **Create Indexes**: Add performance indexes
4. **Deploy Schema**: Deploy to production (backward compatible)

```sql
-- Migration 001: Add hierarchy columns
ALTER TABLE "Page" ADD COLUMN "position" TEXT DEFAULT 'a0';
ALTER TABLE "Page" ADD COLUMN "parentId" TEXT;
ALTER TABLE "Page" ADD COLUMN "icon" TEXT;
ALTER TABLE "Page" ADD COLUMN "coverPhoto" TEXT;
ALTER TABLE "Page" ADD COLUMN "textContent" TEXT;

-- Backfill positions
WITH numbered AS (
  SELECT id, ROW_NUMBER() OVER (ORDER BY "createdAt") as rn
  FROM "Page"
)
UPDATE "Page" p
SET position = CHR(96 + (n.rn / 26)) || (n.rn % 26)
FROM numbered n
WHERE p.id = n.id;

-- Create indexes
CREATE INDEX "Page_parentId_idx" ON "Page"("parentId");
CREATE INDEX "Page_position_idx" ON "Page"("position");
```

### Phase 3: Feature Rollout (Weeks 3-4)
1. **Enable for Admins**: Test with admin users first
2. **Beta Users**: Invite select users to test
3. **Monitor**: Track errors, performance metrics
4. **Iterate**: Fix bugs, optimize queries

### Phase 4: Full Launch (Week 5)
1. **Enable for All**: Remove feature flag
2. **Documentation**: Update user documentation
3. **Training**: Provide guides for new features
4. **Support**: Monitor support tickets

### Rollback Plan
```typescript
// Feature flag check
function useNewPageSystem() {
  const { user } = useAuth();
  return (
    process.env.NEXT_PUBLIC_NEW_PAGES_ENABLED === 'true' ||
    user?.betaFeatures?.includes('new-pages')
  );
}

// In components
const isNewSystem = useNewPageSystem();

return isNewSystem ? (
  <NewPageTree />
) : (
  <LegacyPageList />
);
```

---

## Timeline & Priorities

### Route 1: Minimal Integration (1-2 weeks)
**Priority**: High
**Risk**: Low
**Value**: Medium

| Task | Duration | Dependencies |
|------|----------|--------------|
| Database schema design | 1 day | - |
| Migration scripts | 1 day | Schema |
| Fractional indexing utilities | 1 day | - |
| Page hierarchy queries | 2 days | Migration |
| Page creation/editing UI | 2 days | Queries |
| Tiptap extensions (3-5) | 3 days | - |
| Page metadata UI | 1 day | - |
| Testing & bug fixes | 2 days | All |

**Total**: 10 days

### Route 2: Comprehensive Integration (4-6 weeks)
**Priority**: Medium
**Risk**: Medium
**Value**: High

| Phase | Tasks | Duration |
|-------|-------|----------|
| **Week 1** | Route 1 tasks | 10 days |
| **Week 2** | Storage driver abstraction, attachment system | 5 days |
| **Week 3** | Page history, soft deletes | 5 days |
| **Week 4** | Full-text search, remaining Tiptap extensions | 5 days |
| **Week 5** | Public sharing, advanced features | 5 days |
| **Week 6** | Testing, optimization, documentation | 5 days |

**Total**: 35 days

### Route 3: Full Docmost Architecture (8-12 weeks)
**Priority**: Low
**Risk**: High
**Value**: Very High

| Phase | Tasks | Duration |
|-------|-------|----------|
| **Weeks 1-2** | Route 2 implementation | 10 days |
| **Week 3** | WebSocket server setup (Hocuspocus) | 5 days |
| **Week 4** | Yjs collaboration integration | 5 days |
| **Week 5** | Workspace/Space hierarchy | 5 days |
| **Week 6** | Real-time tree sync | 5 days |
| **Week 7** | Background jobs (BullMQ) | 5 days |
| **Week 8** | Backlinks, permissions | 5 days |
| **Weeks 9-10** | Testing, optimization | 10 days |
| **Weeks 11-12** | Kysely migration (optional) | 10 days |

**Total**: 60 days

---

## Recommendations

### For Immediate Value (Next 2 weeks)
✅ **Implement Route 1**
- Hierarchical pages
- Fractional indexing
- Enhanced attachments
- Key Tiptap extensions (Math, Callouts, Details)

**Why**: Quick wins with minimal risk. Provides immediate organizational benefits without major architectural changes.

### For Production-Ready System (Next 6 weeks)
✅ **Implement Route 2**
- All Route 1 features
- Storage driver abstraction
- Page history & versioning
- Full-text search
- Soft deletes

**Why**: Comprehensive document management with professional features. Good balance of effort vs. value.

### For Future Consideration (3+ months)
⏸️ **Route 3 (Real-time Collaboration)**
- Only if multi-user editing is critical
- Requires significant infrastructure investment
- Higher operational complexity

**Why**: Great feature but high cost. Consider after Route 2 proves value.

---

## Next Steps

### 1. Decision
Choose your integration route based on:
- **Timeline**: How quickly do you need features?
- **Resources**: How much development time is available?
- **Requirements**: Which features are must-haves vs. nice-to-haves?

### 2. Setup
- Create feature branch: `feature/docmost-integration`
- Set up staging environment
- Create database backup

### 3. Implementation
- Follow the detailed plans for your chosen route
- Use feature flags for gradual rollout
- Write tests as you go

### 4. Testing
- Unit tests for utilities (fractional indexing, tree queries)
- Integration tests for page CRUD
- E2E tests for UI workflows
- Load testing for search and tree queries

### 5. Deployment
- Deploy schema changes first (backward compatible)
- Deploy new features behind feature flag
- Gradually enable for users
- Monitor performance and errors

---

## Conclusion

This integration plan provides three clear paths to enhance Scribe with Docmost's document management capabilities:

- **Route 1**: Quick wins for immediate organizational improvements
- **Route 2**: Comprehensive system for production-grade document management
- **Route 3**: Full-featured collaborative platform (future consideration)

I recommend **starting with Route 1** to validate the approach and deliver quick value, then **proceeding to Route 2** for a complete document management system. Route 3 can be considered later if real-time collaboration becomes a priority.

All routes leverage your existing Next.js + Prisma architecture while adopting Docmost's proven patterns for page organization, file handling, and rich editing.

---

**Questions or need clarification on any section?** I'm happy to dive deeper into specific implementation details or create proof-of-concept code for any component.
