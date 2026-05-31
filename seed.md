# ALMANAC
## SEED.md — Spec-Driven Build Contract

Status: Draft v1

Purpose: ensinar um agente de IA a construir o Almanac do zero, preservando as
decisões de produto e engenharia usadas neste projeto.

> Almanac e uma camada social viva sobre documentos HTML e PDF. O usuario cria
> projetos, sobe arquivos, compartilha links e recebe comentarios ancorados no
> ponto exato do documento.

---

## Normative Language

As palavras `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, `RECOMMENDED`,
`MAY` e `OPTIONAL` definem o contrato de implementacao. Um agente que seguir esta
SEED deve tratar cada `MUST` como requisito bloqueante.

`Implementation-defined` significa que a decisao pode variar, mas a implementacao
MUST documentar a escolha no codigo, README ou comentarios operacionais.

---

## 0. How To Use This Seed

Esta SEED e autocontida. Antes de escrever codigo, o agente MUST:

1. Ler este arquivo inteiro.
2. Construir por fases, na ordem da secao 18.
3. Validar cada checkpoint antes de avancar.
4. Usar GitHub como fonte do codigo e Vercel como destino de producao.
5. Usar Supabase para Auth, PostgreSQL e Storage.
6. Usar Google OAuth como unico login obrigatorio da v1.

O projeto esta pronto quando todas as jornadas da secao 17 funcionam em producao.

---

## Summary

- [1. Problem Statement](#1-problem-statement)
- [2. Goals and Non-Goals](#2-goals-and-non-goals)
- [3. System Overview](#3-system-overview)
- [4. Stack and Infrastructure](#4-stack-and-infrastructure)
- [5. Environment Variables](#5-environment-variables)
- [6. Domain Model](#6-domain-model)
- [7. Project Structure](#7-project-structure)
- [8. Database Schema](#8-database-schema)
- [9. RLS Policies](#9-rls-policies)
- [10. Authentication](#10-authentication)
- [11. Storage and Document Versions](#11-storage-and-document-versions)
- [12. Viewer and Anchor System](#12-viewer-and-anchor-system)
- [13. Product Flows](#13-product-flows)
- [14. API Contract](#14-api-contract)
- [15. UI Contract](#15-ui-contract)
- [16. Security and Safety](#16-security-and-safety)
- [17. Acceptance Criteria](#17-acceptance-criteria)
- [18. Build Sequence](#18-build-sequence)

---

## 1. Problem Statement

Revisar documentos ainda e um processo quebrado: feedback se perde em emails,
prints, planilhas, mensagens soltas e comentarios sem contexto. Ferramentas de
edicao colaborativa resolvem escrita conjunta, mas nao resolvem leitura, revisao
e entendimento coletivo em cima de documentos ja existentes.

Almanac resolve esse problema criando uma camada de comentarios ancorados sobre
arquivos HTML e PDF. O autor sobe documentos dentro de um projeto, compartilha um
link, e revisores deixam comentarios exatamente onde a discussao acontece.

Boundary importante:

- Almanac e uma ferramenta de revisao contextual, nao um editor de texto.
- O arquivo original MUST permanecer no Supabase Storage.
- Comentarios e reacoes vivem no banco, separados do documento.
- O link publico MAY ser lido sem login, mas comentar MUST exigir autenticacao.

---

## 2. Goals and Non-Goals

### 2.1 Goals

- Permitir login com Google via Supabase Auth.
- Permitir que usuarios criem projetos.
- Permitir upload de multiplos documentos HTML/PDF dentro de um projeto.
- Renderizar HTML e PDF no browser.
- Permitir comentarios com pins posicionais.
- Persistir pins e comentarios por documento e por versao.
- Permitir threads, reacoes e resolucao de comentarios.
- Permitir share link publico por documento.
- Mostrar navegacao entre arquivos do mesmo projeto em links compartilhados.
- Usar GitHub para versionamento e Vercel para deploy.
- Manter uma arquitetura simples o suficiente para hackathon, mas robusta o
  bastante para producao inicial.

### 2.2 Non-Goals

- Edicao colaborativa do documento dentro do app.
- Sistema multi-tenant com organizacoes e billing.
- Permissoes granulares por equipe.
- OCR ou extracao perfeita de texto de PDF.
- Comentarios por selecao de texto como requisito obrigatorio.
- IA generativa como requisito da v1. Insights de IA MAY ser adicionados depois.

---

## 3. System Overview

### 3.1 Main Components

1. `Next.js App Router`
   - Renderiza paginas, server components e API routes.
   - Protege rotas autenticadas via middleware Supabase.

2. `Supabase Auth`
   - Gerencia sessoes.
   - Usa Google OAuth.
   - Cria `profiles` automaticamente via trigger.

3. `Supabase PostgreSQL`
   - Guarda projetos, documentos, versoes, comentarios, reacoes e atividades.
   - Usa RLS para garantir isolamento por usuario.

4. `Supabase Storage`
   - Guarda HTML/PDF por usuario, documento e versao.
   - Serve arquivos via signed URLs ou public URLs conforme configuracao.

5. `Document Viewer`
   - Orquestra HTMLViewer ou PDFViewer.
   - Mantem painel de comentarios e PinLayer.
   - Calcula coordenadas normalizadas para pins.

6. `Share Surface`
   - Rota `/shared/[token]`.
   - Busca documento por `share_token`.
   - Mostra comentarios e permite login para comentar.

7. `GitHub + Vercel`
   - GitHub MUST conter o repo principal.
   - Vercel MUST estar conectado ao repo para deploy automatico.
   - Variaveis de ambiente MUST ser configuradas na Vercel.

### 3.2 Architectural Layers

1. Product layer: projetos, documentos, revisao e compartilhamento.
2. Auth layer: Supabase Auth + Google OAuth + middleware.
3. Data layer: Supabase PostgreSQL + RLS + mappers TypeScript.
4. Storage layer: buckets, upload, versoes e signed URLs.
5. Viewer layer: renderizacao, scroll, pins e painel.
6. API layer: contratos HTTP para mutacoes e leituras.
7. Deployment layer: GitHub, Vercel, env vars e smoke tests.

---

## 4. Stack and Infrastructure

### 4.1 Required Stack

| Layer | Technology |
| --- | --- |
| Framework | Next.js 14 App Router |
| Language | TypeScript |
| UI | React 18 |
| Styling | Tailwind CSS |
| Components | shadcn/ui |
| Icons | lucide-react |
| Auth | Supabase Auth |
| OAuth Provider | Google |
| Database | Supabase PostgreSQL |
| Storage | Supabase Storage |
| Deploy | Vercel |
| Source Control | GitHub |

### 4.2 Recommended Dependencies

```json
{
  "@supabase/ssr": "^0.5.0",
  "@supabase/supabase-js": "^2.45.0",
  "next": "^14.2.0",
  "react": "^18.0.0",
  "react-dom": "^18.0.0",
  "pdfjs-dist": "^4.4.168",
  "lucide-react": "^0.383.0",
  "sonner": "^1.5.0",
  "zod": "^3.23.0",
  "react-hook-form": "^7.53.0",
  "zustand": "^4.5.0",
  "tailwind-merge": "^2.3.0",
  "clsx": "^2.1.0",
  "class-variance-authority": "^0.7.0"
}
```

### 4.3 Runtime Requirements

- Node.js MUST be 18.17+.
- The app MUST build with `npm run build`.
- Vercel deployment MUST use the same environment variables as local dev.
- Supabase project MUST have Google provider enabled.

---

## 5. Environment Variables

```env
NEXT_PUBLIC_SUPABASE_URL=https://[project-ref].supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=[anon_key]
SUPABASE_SERVICE_ROLE_KEY=[service_role_key]
NEXT_PUBLIC_APP_URL=https://[vercel-domain].vercel.app
```

Rules:

- `NEXT_PUBLIC_*` MAY be used in client components.
- `SUPABASE_SERVICE_ROLE_KEY` MUST NOT be imported by client components.
- Admin operations MUST happen only in API routes or server-side helpers.
- Vercel MUST define all env vars for Production and Preview.
- `.env.local` MUST NOT be committed to GitHub.

---

## 6. Domain Model

### 6.1 User Profile

`profiles` mirrors `auth.users`.

Fields:

- `id`: UUID, same as `auth.users.id`.
- `email`: text.
- `full_name`: text nullable.
- `avatar_url`: text nullable.
- `created_at`: timestamp.

### 6.2 Project

A project groups related documents.

Fields:

- `id`: UUID.
- `owner_id`: auth user UUID.
- `title`: required.
- `description`: optional.
- `status`: `active` or `finalized`.
- `document_count`: denormalized integer.
- `created_at`, `updated_at`.

### 6.3 Document

A document is one uploaded HTML or PDF file.

Fields:

- `id`: UUID.
- `project_id`: optional UUID.
- `owner_id`: auth user UUID.
- `title`: required.
- `description`: optional.
- `type`: `html` or `pdf`.
- `storage_path`: latest file path.
- `public_url`: latest public or signed URL reference.
- `status`: `active`, `archived`, or `published`.
- `share_token`: unique public token.
- `is_public`: boolean.
- `comment_count`: denormalized integer.
- `view_count`: denormalized integer.
- `created_at`, `updated_at`.

### 6.4 Document Version

Every upload creates a version. Comments SHOULD be tied to the version where they
were created, so feedback does not silently drift when a file changes.

Fields:

- `id`: UUID.
- `document_id`: UUID.
- `version_number`: integer, unique per document.
- `storage_path`: required.
- `public_url`: required.
- `title`: optional.
- `change_summary`: optional.
- `created_by`: auth user UUID.
- `created_at`.

### 6.5 Comment

Root comments have anchors. Replies do not.

Fields:

- `id`: UUID.
- `document_id`: UUID.
- `document_version_id`: nullable UUID.
- `author_id`: auth user UUID.
- `parent_id`: nullable UUID.
- `anchor_type`: `point` or `selection`, nullable for replies.
- `anchor_x`, `anchor_y`: percent coordinates from 0 to 100.
- `anchor_x_px`, `anchor_y_px`: optional pixel coordinates.
- `page_number`: optional PDF page.
- `body`: required.
- `status`: `open` or `resolved`.
- `is_deleted`: boolean.
- `reaction_count`, `reply_count`.
- `created_at`, `updated_at`.

### 6.6 Reaction

A reaction is one emoji by one user on one comment.

Allowed emojis SHOULD be: `👍`, `❤️`, `🔥`, `🤔`, `💡`, `😮`.

### 6.7 Activity Feed

Activity feed stores useful product events. It is not required for the first UI,
but SHOULD be implemented because it helps audit collaboration.

---

## 7. Project Structure

The agent SHOULD create this structure:

```txt
almanac/
├── app/
│   ├── api/
│   │   ├── comments/route.ts
│   │   ├── comments/[id]/route.ts
│   │   ├── documents/route.ts
│   │   ├── documents/[id]/route.ts
│   │   ├── documents/[id]/comments/route.ts
│   │   ├── documents/[id]/upload/route.ts
│   │   ├── documents/[id]/versions/route.ts
│   │   ├── projects/route.ts
│   │   ├── projects/[id]/route.ts
│   │   ├── projects/[id]/upload/route.ts
│   │   ├── reactions/route.ts
│   │   └── share/[token]/route.ts
│   ├── auth/callback/route.ts
│   ├── dashboard/page.tsx
│   ├── documents/[id]/page.tsx
│   ├── documents/new/page.tsx
│   ├── projects/[id]/page.tsx
│   ├── projects/new/page.tsx
│   ├── shared/[token]/page.tsx
│   ├── globals.css
│   ├── layout.tsx
│   └── page.tsx
├── components/
│   ├── auth/
│   ├── comments/
│   ├── dashboard/
│   ├── documents/
│   ├── layout/
│   ├── projects/
│   ├── ui/
│   └── viewer/
├── hooks/
├── lib/
│   ├── supabase/
│   │   ├── admin.ts
│   │   ├── client.ts
│   │   ├── middleware.ts
│   │   └── server.ts
│   ├── anchors.ts
│   ├── api.ts
│   ├── authors.ts
│   ├── document-versions.ts
│   ├── mappers.ts
│   ├── sanitize.ts
│   ├── types.ts
│   └── utils.ts
├── supabase/migrations/
├── middleware.ts
├── package.json
└── README.md
```

---

## 8. Database Schema

Execute in Supabase SQL Editor or migrate with Supabase CLI.

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users ON DELETE CASCADE,
  email TEXT NOT NULL,
  full_name TEXT,
  avatar_url TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO profiles (id, email, full_name, avatar_url)
  VALUES (
    NEW.id,
    NEW.email,
    NEW.raw_user_meta_data->>'full_name',
    NEW.raw_user_meta_data->>'avatar_url'
  )
  ON CONFLICT (id) DO NOTHING;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();

CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  title TEXT NOT NULL CHECK (length(title) > 0 AND length(title) <= 200),
  description TEXT,
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'finalized')),
  document_count INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_owner_id ON projects(owner_id);

CREATE TRIGGER projects_updated_at
  BEFORE UPDATE ON projects
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE SET NULL,
  owner_id UUID REFERENCES auth.users ON DELETE CASCADE NOT NULL,
  title TEXT NOT NULL CHECK (length(title) > 0 AND length(title) <= 200),
  description TEXT,
  type TEXT NOT NULL CHECK (type IN ('html', 'pdf')),
  storage_path TEXT,
  public_url TEXT,
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'archived', 'published')),
  share_token TEXT UNIQUE DEFAULT encode(gen_random_bytes(16), 'hex'),
  is_public BOOLEAN DEFAULT TRUE,
  comment_count INT DEFAULT 0,
  view_count INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_documents_owner ON documents(owner_id);
CREATE INDEX idx_documents_project_id ON documents(project_id);
CREATE INDEX idx_documents_share_token ON documents(share_token);

CREATE TRIGGER documents_updated_at
  BEFORE UPDATE ON documents
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TABLE document_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID REFERENCES documents ON DELETE CASCADE NOT NULL,
  version_number INT NOT NULL,
  storage_path TEXT NOT NULL,
  public_url TEXT NOT NULL,
  title TEXT,
  change_summary TEXT,
  created_by UUID REFERENCES auth.users NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(document_id, version_number)
);

CREATE INDEX idx_versions_document ON document_versions(document_id);

CREATE TABLE comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID REFERENCES documents ON DELETE CASCADE NOT NULL,
  document_version_id UUID REFERENCES document_versions(id) ON DELETE SET NULL,
  author_id UUID REFERENCES auth.users NOT NULL,
  parent_id UUID REFERENCES comments(id) ON DELETE CASCADE,
  anchor_type TEXT CHECK (anchor_type IN ('point', 'selection')),
  anchor_x FLOAT CHECK (anchor_x >= 0 AND anchor_x <= 100),
  anchor_y FLOAT CHECK (anchor_y >= 0 AND anchor_y <= 100),
  anchor_x_px DOUBLE PRECISION,
  anchor_y_px DOUBLE PRECISION,
  anchor_quote TEXT,
  anchor_selector TEXT,
  anchor_range_start INT,
  anchor_range_end INT,
  page_number INT,
  body TEXT NOT NULL CHECK (length(body) > 0 AND length(body) <= 2000),
  status TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'resolved')),
  is_deleted BOOLEAN DEFAULT FALSE,
  reaction_count INT DEFAULT 0,
  reply_count INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  CONSTRAINT root_comment_needs_anchor CHECK (
    parent_id IS NOT NULL OR anchor_type IS NOT NULL
  )
);

CREATE INDEX idx_comments_document ON comments(document_id);
CREATE INDEX idx_comments_version ON comments(document_version_id);
CREATE INDEX idx_comments_author ON comments(author_id);
CREATE INDEX idx_comments_parent ON comments(parent_id);

CREATE TRIGGER comments_updated_at
  BEFORE UPDATE ON comments
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TABLE reactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  comment_id UUID REFERENCES comments ON DELETE CASCADE NOT NULL,
  user_id UUID REFERENCES auth.users NOT NULL,
  emoji TEXT NOT NULL CHECK (emoji IN ('👍','❤️','🔥','🤔','💡','😮')),
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(comment_id, user_id, emoji)
);

CREATE INDEX idx_reactions_comment ON reactions(comment_id);

CREATE TABLE activity_feed (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID REFERENCES documents ON DELETE CASCADE NOT NULL,
  actor_id UUID REFERENCES auth.users NOT NULL,
  action TEXT NOT NULL,
  payload JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_activity_document ON activity_feed(document_id);
CREATE INDEX idx_activity_created ON activity_feed(created_at DESC);

CREATE OR REPLACE FUNCTION sync_project_on_document_change()
RETURNS TRIGGER AS $$
DECLARE
  affected_id UUID;
BEGIN
  IF TG_OP = 'INSERT' THEN
    affected_id := NEW.project_id;
  ELSIF TG_OP = 'DELETE' THEN
    affected_id := OLD.project_id;
  ELSIF TG_OP = 'UPDATE' THEN
    IF OLD.project_id IS DISTINCT FROM NEW.project_id THEN
      UPDATE projects
      SET document_count = (SELECT count(*) FROM documents WHERE project_id = OLD.project_id),
          updated_at = now()
      WHERE id = OLD.project_id;
    END IF;
    affected_id := NEW.project_id;
  END IF;

  IF affected_id IS NOT NULL THEN
    UPDATE projects
    SET document_count = (SELECT count(*) FROM documents WHERE project_id = affected_id),
        updated_at = now()
    WHERE id = affected_id;
  END IF;

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER on_document_change_sync_project
  AFTER INSERT OR UPDATE OR DELETE ON documents
  FOR EACH ROW EXECUTE FUNCTION sync_project_on_document_change();

CREATE OR REPLACE FUNCTION update_comment_count()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' AND NEW.parent_id IS NULL THEN
    UPDATE documents SET comment_count = comment_count + 1 WHERE id = NEW.document_id;
  ELSIF TG_OP = 'DELETE' AND OLD.parent_id IS NULL THEN
    UPDATE documents SET comment_count = GREATEST(0, comment_count - 1) WHERE id = OLD.document_id;
  END IF;
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER on_comment_change
  AFTER INSERT OR DELETE ON comments
  FOR EACH ROW EXECUTE FUNCTION update_comment_count();

CREATE OR REPLACE FUNCTION update_reaction_count()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE comments SET reaction_count = reaction_count + 1 WHERE id = NEW.comment_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE comments SET reaction_count = GREATEST(0, reaction_count - 1) WHERE id = OLD.comment_id;
  END IF;
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER on_reaction_change
  AFTER INSERT OR DELETE ON reactions
  FOR EACH ROW EXECUTE FUNCTION update_reaction_count();
```

---

## 9. RLS Policies

```sql
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE document_versions ENABLE ROW LEVEL SECURITY;
ALTER TABLE comments ENABLE ROW LEVEL SECURITY;
ALTER TABLE reactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE activity_feed ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Profiles are readable"
  ON profiles FOR SELECT USING (true);

CREATE POLICY "User updates own profile"
  ON profiles FOR UPDATE USING (auth.uid() = id);

CREATE POLICY "Users view own projects"
  ON projects FOR SELECT USING (auth.uid() = owner_id);

CREATE POLICY "Users create own projects"
  ON projects FOR INSERT WITH CHECK (auth.uid() = owner_id);

CREATE POLICY "Users update own projects"
  ON projects FOR UPDATE USING (auth.uid() = owner_id);

CREATE POLICY "Users delete own projects"
  ON projects FOR DELETE USING (auth.uid() = owner_id);

CREATE POLICY "Owner manages own documents"
  ON documents FOR ALL USING (auth.uid() = owner_id);

CREATE POLICY "Public documents are readable"
  ON documents FOR SELECT USING (is_public = true);

CREATE POLICY "Owner manages versions"
  ON document_versions FOR ALL USING (
    EXISTS (SELECT 1 FROM documents WHERE documents.id = document_id AND documents.owner_id = auth.uid())
  );

CREATE POLICY "Public document versions are readable"
  ON document_versions FOR SELECT USING (
    EXISTS (SELECT 1 FROM documents WHERE documents.id = document_id AND documents.is_public = true)
  );

CREATE POLICY "Accessible comments are readable"
  ON comments FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM documents
      WHERE documents.id = comments.document_id
      AND (documents.is_public = true OR documents.owner_id = auth.uid())
    )
  );

CREATE POLICY "Authenticated users create comments in accessible docs"
  ON comments FOR INSERT WITH CHECK (
    auth.uid() = author_id AND
    EXISTS (
      SELECT 1 FROM documents
      WHERE documents.id = document_id
      AND (documents.is_public = true OR documents.owner_id = auth.uid())
    )
  );

CREATE POLICY "Author updates own comments"
  ON comments FOR UPDATE USING (auth.uid() = author_id);

CREATE POLICY "Author or owner deletes comments"
  ON comments FOR DELETE USING (
    auth.uid() = author_id OR
    EXISTS (SELECT 1 FROM documents WHERE documents.id = document_id AND documents.owner_id = auth.uid())
  );

CREATE POLICY "Reactions are readable"
  ON reactions FOR SELECT USING (true);

CREATE POLICY "Users manage own reactions"
  ON reactions FOR ALL USING (auth.uid() = user_id);

CREATE POLICY "Readable activity for accessible docs"
  ON activity_feed FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM documents
      WHERE documents.id = document_id
      AND (documents.is_public = true OR documents.owner_id = auth.uid())
    )
  );

CREATE POLICY "Authenticated users insert own activity"
  ON activity_feed FOR INSERT WITH CHECK (auth.uid() = actor_id);
```

---

## 10. Authentication

### 10.1 Supabase Google OAuth

The implementation MUST:

1. Enable Google provider in Supabase.
2. Create OAuth client in Google Cloud Console.
3. Add authorized redirect URI:
   `https://[project-ref].supabase.co/auth/v1/callback`.
4. Add the site URL in Supabase Auth URL configuration.
5. Add Vercel production and preview callback URLs as allowed redirects.

### 10.2 Client Helpers

`lib/supabase/client.ts` MUST use `createBrowserClient`.

`lib/supabase/server.ts` MUST use `createServerClient` with cookies from
`next/headers`.

`lib/supabase/admin.ts` MUST create a service-role client and MUST only be imported
from server files.

### 10.3 Middleware

`middleware.ts` MUST protect:

- `/dashboard`
- `/documents`
- `/projects`

`/shared/[token]` MUST remain accessible without auth.

---

## 11. Storage and Document Versions

### 11.1 Bucket

The app MUST use a Supabase Storage bucket named `documents`.

Recommended path:

```txt
{owner_id}/{document_id}/versions/v{version_number}.{html|pdf}
```

### 11.2 Upload Contract

On upload:

1. Validate authenticated user.
2. Validate ownership of project/document.
3. Create document row if upload happens inside project.
4. Compute next `version_number`.
5. Upload file to Storage.
6. Insert row in `document_versions`.
7. Update `documents.storage_path` and `documents.public_url`.
8. Return mapped document/version data.

### 11.3 HTML Safety

HTML rendering MUST NOT execute arbitrary script directly in the main app document.

Implementation SHOULD:

- sanitize HTML server-side or before rendering;
- remove `<script>` tags and inline event handlers at minimum;
- render in an iframe or isolated viewer surface;
- prefer signed URLs for private buckets.

---

## 12. Viewer and Anchor System

### 12.1 Viewer Contract

`DocumentViewer` MUST own:

- current comment mode;
- selected comment id;
- local comment list;
- pending anchor;
- active document version;
- HTML scroll metrics when rendering HTML.

It MUST render:

- `HTMLViewer` for HTML;
- `PDFViewer` for PDF;
- `PinLayer` over the document;
- `CommentPanel` beside the document on desktop and below on mobile.

### 12.2 Point Anchors

The v1 anchor type is `point`.

Coordinates MUST be normalized:

- `x`: percent of viewport/container width.
- `y`: percent of full document height for HTML.
- `y`: percent of rendered page/container height for basic PDF.
- values MUST be clamped between 0 and 100.

Example helper:

```ts
export interface PointAnchor {
  type: "point";
  x: number;
  y: number;
  pageNumber?: number;
}

export function clampPercent(value: number) {
  return Math.min(100, Math.max(0, value));
}
```

### 12.3 HTML Metrics

For HTML, pins MUST remain visually attached while scrolling. `HTMLViewer` SHOULD
report:

- `scrollTop`;
- `scrollHeight`;
- `viewportHeight`.

`PinLayer` SHOULD convert stored document coordinates into visible viewport
coordinates. Off-screen pins MAY be hidden when outside `-5..105`.

### 12.4 Pending Pin

When comment mode is active:

1. User clicks document.
2. App calculates anchor.
3. App shows yellow pending pin.
4. CommentPanel asks for body text.
5. Save creates comment via `/api/comments`.
6. Cancel removes pending pin.

### 12.5 Comment Pin Visual

Pins SHOULD:

- be circular;
- show index number;
- be blue when open;
- be green when resolved;
- be yellow when pending;
- show selected state with border/shadow;
- remain stable in size across states.

---

## 13. Product Flows

### 13.1 Landing

`/` MUST show:

- Almanac brand;
- concise value proposition;
- Google login button;
- explanation of upload, comment, share and review flow.

### 13.2 Dashboard

`/dashboard` MUST show:

- authenticated user's projects;
- empty state with CTA;
- link to create project;
- cards with title, description, document count and updated date.

### 13.3 Project Detail

`/projects/[id]` MUST show:

- project title and description;
- list of documents;
- upload form for `.html` and `.pdf`;
- delete project action for owner.

### 13.4 Document Detail

`/documents/[id]` MUST show:

- document viewer;
- comments;
- share button;
- project file navigation when opened from a project;
- version list or latest version state.

### 13.5 Shared Document

`/shared/[token]` MUST:

- work without a session;
- fetch only `is_public = true` documents;
- show current document and comments;
- show sibling documents in the same project when public;
- allow logged-in users to comment;
- show login prompt for anonymous users who want to comment.

---

## 14. API Contract

All API routes MUST:

- validate auth when mutation or private read is required;
- return JSON;
- use appropriate HTTP statuses;
- avoid leaking service-role errors to users when possible;
- use `try/catch` for mutating routes;
- map database rows into stable TypeScript response shapes.

### 14.1 Projects

`GET /api/projects`

- Returns authenticated user's projects.

`POST /api/projects`

- Body: `{ title: string, description?: string }`
- Creates project for current user.

`PATCH /api/projects/[id]`

- Updates title/description/status.
- Owner only.

`DELETE /api/projects/[id]`

- Deletes project.
- Owner only.

`POST /api/projects/[id]/upload`

- Body: `FormData(file)`
- Creates document inside project and first version.

### 14.2 Documents

`GET /api/documents`

- Returns current user's documents.

`POST /api/documents`

- Body: `{ title, description?, type, projectId? }`
- Creates document row.

`GET /api/documents/[id]`

- Returns document if owner or public.

`PATCH /api/documents/[id]`

- Owner only.
- Updates metadata/status/public flag.

`DELETE /api/documents/[id]`

- Owner only.

`POST /api/documents/[id]/upload`

- Uploads a new version.

`GET /api/documents/[id]/comments`

- Returns non-deleted comments for latest or requested version.

`GET /api/documents/[id]/versions`

- Returns versions ordered by version number desc.

### 14.3 Comments

`POST /api/comments`

Body:

```ts
{
  documentId: string;
  versionId?: string | null;
  parentId?: string | null;
  body: string;
  anchor?: { x: number; y: number };
  pageNumber?: number;
}
```

Rules:

- Root comment MUST include anchor.
- Reply MUST include parentId and MUST NOT require anchor.
- User MUST be authenticated.
- Document MUST be accessible.

`PATCH /api/comments/[id]`

- Author MAY edit body.
- Author MAY change own status.
- Owner MAY resolve comments on own documents.

`DELETE /api/comments/[id]`

- Soft delete by setting `is_deleted = true`.

### 14.4 Reactions

`POST /api/reactions`

- Body: `{ commentId: string, emoji: string }`
- Toggles user's reaction.

### 14.5 Share

`GET /api/share/[token]`

- Public route.
- Returns document and comments only if document is public.

---

## 15. UI Contract

### 15.1 Design Direction

Almanac SHOULD feel editorial, precise and calm. It is a document workspace, not
a marketing site.

Visual rules:

- Use restrained neutral backgrounds.
- Use black/zinc/stone as base.
- Use one accent color for active states.
- Use 8px radius for cards and panels.
- Use lucide icons in buttons.
- Avoid oversized decorative sections inside the app.
- Avoid explanatory in-app text where a familiar control is enough.

### 15.2 Responsive Behavior

Desktop:

- Viewer left, comment panel right.
- Navbar fixed or visually persistent.
- Project file navigation under viewer when present.

Mobile:

- Viewer top, comments bottom.
- Comment controls MUST remain reachable.
- Text MUST not overflow buttons or cards.

### 15.3 Required Components

- `GoogleSignInButton`
- `Navbar`
- `ProjectList`
- `ProjectCard`
- `NewProjectForm`
- `ProjectUploadForm`
- `ProjectDocuments`
- `DocumentViewer`
- `HTMLViewer`
- `PDFViewer`
- `PinLayer`
- `CommentPin`
- `CommentPanel`
- `CommentCard`
- `ShareButton`
- `DocumentVersions`

---

## 16. Security and Safety

The implementation MUST:

1. Keep service role key server-only.
2. Use RLS on all user data tables.
3. Validate document ownership before admin writes.
4. Validate file type and size before upload.
5. Sanitize HTML before rendering.
6. Keep `/shared/[token]` read-only for anonymous users.
7. Require auth for comments, replies and reactions.
8. Clamp anchor coordinates before writing to DB.
9. Never trust client-provided `author_id` or `owner_id`.
10. Avoid committing `.env.local`.

The implementation SHOULD:

- use signed URLs if Storage bucket is private;
- avoid exposing raw Supabase errors in production;
- include smoke tests before deploy;
- run `npm run build` before pushing to GitHub.

---

## 17. Acceptance Criteria

### Journey 1 — Google Login

- [ ] User opens `/`.
- [ ] User clicks Google login.
- [ ] Supabase OAuth completes.
- [ ] User lands on `/dashboard`.
- [ ] Row exists in `profiles`.

### Journey 2 — Create Project

- [ ] User opens dashboard.
- [ ] User clicks "Novo projeto".
- [ ] User creates project with title.
- [ ] Project appears on dashboard.

### Journey 3 — Upload Document

- [ ] User opens project.
- [ ] User uploads `.html`.
- [ ] Document row is created.
- [ ] Version row is created.
- [ ] Storage file exists.
- [ ] Document appears in project list.

### Journey 4 — Render and Comment

- [ ] User opens document.
- [ ] HTML/PDF renders.
- [ ] User activates comment mode.
- [ ] User clicks document.
- [ ] Pending pin appears.
- [ ] User writes comment.
- [ ] Permanent pin appears.
- [ ] Reload preserves pin position.

### Journey 5 — Thread, React and Resolve

- [ ] User replies to a comment.
- [ ] Reply appears under parent.
- [ ] User toggles reaction.
- [ ] Reaction count updates.
- [ ] User resolves comment.
- [ ] Pin changes resolved state.

### Journey 6 — Share Link

- [ ] Owner copies share link.
- [ ] Anonymous visitor opens `/shared/[token]`.
- [ ] Visitor sees document and comments.
- [ ] Anonymous visitor cannot comment directly.
- [ ] Logged-in visitor can comment.

### Journey 7 — Production Deploy

- [ ] Repo exists on GitHub.
- [ ] Vercel project is connected to repo.
- [ ] Env vars are configured in Vercel.
- [ ] Production build succeeds.
- [ ] Journeys 1-6 pass on production URL.

---

## 18. Build Sequence

Follow this exact order.

### Phase 1 — Foundation

1. Create Next.js app with TypeScript, App Router and Tailwind.
2. Install dependencies.
3. Initialize shadcn/ui.
4. Create `lib/utils.ts`.
5. Create shared UI components.
6. Create basic layout, globals, loading, error and not-found pages.

Checkpoint: `npm run dev` starts.

### Phase 2 — Supabase Base

1. Create Supabase project.
2. Create tables, triggers and RLS policies.
3. Create Storage bucket `documents`.
4. Add `.env.local`.
5. Create Supabase helpers: `client`, `server`, `admin`, `middleware`.

Checkpoint: authenticated server read works locally.

### Phase 3 — Auth

1. Configure Google OAuth in Supabase and Google Cloud.
2. Implement `/auth/callback`.
3. Implement middleware protection.
4. Implement Google sign-in button.
5. Implement landing page.

Checkpoint: Journey 1 passes.

### Phase 4 — Projects

1. Implement project API routes.
2. Implement dashboard project list.
3. Implement new project page/form.
4. Implement project detail page.

Checkpoint: Journey 2 passes.

### Phase 5 — Upload and Versions

1. Implement `uploadDocumentVersion`.
2. Implement project upload route.
3. Implement document upload route for new versions.
4. Implement version list.
5. Wire project document list.

Checkpoint: Journey 3 passes.

### Phase 6 — Viewer

1. Implement `HTMLViewer`.
2. Implement `PDFViewer`.
3. Implement `DocumentViewer`.
4. Fetch latest version and file content.
5. Render document page.

Checkpoint: document opens and renders.

### Phase 7 — Anchor System

1. Implement anchor helpers.
2. Implement `PinLayer`.
3. Implement `CommentPin`.
4. Implement pending pin flow.
5. Implement `/api/comments`.
6. Implement `CommentPanel`.

Checkpoint: Journey 4 passes.

### Phase 8 — Collaboration Features

1. Implement replies.
2. Implement comment status update.
3. Implement soft delete.
4. Implement reactions.
5. Add sonner toasts.

Checkpoint: Journey 5 passes.

### Phase 9 — Sharing

1. Implement share API.
2. Implement shared page.
3. Implement `ShareButton`.
4. Add project sibling navigation in shared view.
5. Test anonymous and logged-in modes.

Checkpoint: Journey 6 passes.

### Phase 10 — GitHub and Vercel

1. Initialize git.
2. Commit clean app.
3. Push to GitHub.
4. Import repo in Vercel.
5. Configure env vars.
6. Deploy.
7. Run production smoke test.

Checkpoint: Journey 7 passes.

---

## 19. Agent Operating Rules

When building Almanac, the agent MUST:

1. Prefer working code over speculative abstraction.
2. Keep all business rules close to the route or component that owns them.
3. Use TypeScript types in `lib/types.ts` and mappers in `lib/mappers.ts`.
4. Avoid client imports of server-only Supabase admin code.
5. Keep comments version-aware.
6. Test anonymous share links in a clean session.
7. Run `npm run build` before declaring completion.
8. Document any implementation-defined deviation from this SEED.

Principle of the product:

> O documento continua sendo o artefato central. Almanac adiciona conversa,
> contexto e memoria coletiva sem destruir o arquivo original.

---

Almanac v1.0 — SEED.md closed enough to build, open enough to evolve.
