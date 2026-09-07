---
layout: single
title: "AI와 함께 개발하는 팀의 프로젝트 세팅: 실행 환경부터 리뷰·배포까지"
categories:
  - "AI"
tags:
  - "AI 코딩"
  - "프로젝트 세팅"
  - "팀 개발"
  - "AGENTS.md"
  - "Next.js"
  - "CI/CD"
toc: true
author_profile: true
sidebar:
  nav: "docs"
---

혼자 AI와 코드를 만들 때는 대화 속에서 규칙을 맞출 수 있습니다. 하지만 여러 사람이 각자의 AI와 작업하면 상황이 달라집니다. 한 사람은 API에 SQL을 직접 넣고, 다른 사람은 별도 계층을 만들고, 또 다른 사람은 테스트를 생략할 수 있습니다.

유지보수 가능한 팀 프로젝트에는 **대화 밖에 남는 규칙, 누구나 재현할 수 있는 환경, 자동으로 확인하는 완료 기준**이 필요합니다. AI가 바뀌거나 담당자가 바뀌어도 같은 방식으로 작업을 이어갈 수 있어야 합니다.

이 글은 **AI 코딩 도구를 사용하는 여러 개발자가 함께 만드는 웹 서비스의 초기 프로젝트**를 구축합니다. 서비스 내부에서 LLM을 호출하는 기능이나 여러 AI 에이전트를 운영하는 플랫폼을 만드는 글은 아닙니다.

아래 파일과 명령을 순서대로 적용하면 PostgreSQL에 작업을 저장·조회하는 Next.js API, 입력 검증, 단위 테스트, 실제 API 테스트, CI와 컨테이너 실행 환경을 만들 수 있습니다. 업무용 로그인과 권한 모델은 서비스 요구에 따라 추가할 영역이며, 여기서는 로컬 개발용 API 키를 사용합니다.

## 완성할 구조와 책임을 먼저 정한다

기준 환경은 **Node.js 24, npm, Next.js 16, TypeScript 5, PostgreSQL 17, Docker Compose v2, GitHub Actions**입니다. macOS·Linux 또는 Windows의 WSL 환경에서 명령을 실행하는 것으로 가정합니다. Node.js와 Docker, Git은 미리 설치되어 있어야 합니다.

작은 팀이 시작하기 쉽도록 하나의 저장소와 하나의 애플리케이션으로 구성합니다. 배포 단위를 늘리기 전에 코드 안에서 책임을 나눕니다.

~~~text
HTTP 요청
  → route: 인증·입출력·상태 코드
  → schema: 입력 계약과 검증
  → repository: SQL과 저장·조회
  → PostgreSQL

사람: 요구사항·설계 판단·리뷰·배포 책임
AI: 정해진 범위의 구현·검증·변경 설명
CI: 누구의 코드든 같은 검사 실행
~~~

복잡한 업무 규칙이 생기면 route와 repository 사이에 service 계층을 추가합니다. 처음부터 빈 계층을 많이 만드는 대신 실제 책임이 생겼을 때 분리합니다.

## 1. 빈 저장소와 공통 명령 만들기

아래 명령은 **블로그 저장소가 아닌 새 프로젝트 폴더**에서 실행합니다.

~~~bash
mkdir ai-team-starter
cd ai-team-starter
git init -b main
mkdir -p src/app/api/health src/app/api/tasks src/features/tasks src/lib
mkdir -p migrations tests/unit tests/api .github/workflows docs/adr docs/tasks
~~~

이후의 `파일:`에 표시한 경로에 내용을 저장합니다. 중간에 생략된 애플리케이션 파일은 없습니다.

**파일: `package.json`**

~~~json
{
  "name": "ai-team-starter",
  "version": "0.1.0",
  "private": true,
  "engines": { "node": ">=24 <25" },
  "scripts": {
    "dev": "next dev --hostname 127.0.0.1",
    "build": "next build",
    "start": "next start --hostname 127.0.0.1",
    "lint": "eslint . --max-warnings=0",
    "typecheck": "next typegen && tsc --noEmit",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "test": "vitest run",
    "test:api": "dotenv -e .env -- playwright test",
    "db:migrate": "dotenv -e .env -- node-pg-migrate up",
    "check": "npm run format:check && npm run lint && npm run typecheck && npm test && npm run build"
  }
}
~~~

次に依存関係をインストールします。

~~~bash
npm install --save-exact next@16.3.4 react@19.2.8 react-dom@19.2.8 pg@8.23.0 zod@4.5.4 server-only@0.0.1
npm install --save-dev --save-exact typescript@5 @types/node@24 @types/react@19 @types/react-dom@19 @types/pg@8 eslint@9 eslint-config-next@16.3.4 vitest@4 @playwright/test@1.63.0 node-pg-migrate@9 dotenv-cli@8 prettier@3
node -p 'process.versions.node' > .nvmrc
~~~

처음 설치하는 담당자가 버전을 확정하고 **`package.json`, `package-lock.json`, `.nvmrc`를 함께 커밋**합니다. 이후 팀원과 CI는 `npm install` 대신 `npm ci`로 동일한 잠금 파일을 사용합니다. 패키지 업데이트는 별도 PR로 검증합니다.

여기서 메이저 버전만 지정한 개발 도구도 `--save-exact`로 설치된 실제 버전이 기록됩니다. 장기적인 재현성의 기준은 위 명령을 매번 다시 실행하는 것이 아니라, 팀이 검증해 커밋한 잠금 파일입니다. 실행 시점의 지원 범위는 [Next.js 설치 문서](https://nextjs.org/docs/app/getting-started/installation)와 [Node.js 릴리스 안내](https://nodejs.org/en/about/previous-releases)를 확인합니다.

### 환경별로 달라지면 안 되는 설정

**파일: `.gitignore`**

~~~text
node_modules/
.next/
coverage/
playwright-report/
test-results/
*.tsbuildinfo
.env*
!.env.example
.DS_Store
~~~

**파일: `.editorconfig`**

~~~ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false
~~~

**파일: `.prettierignore`**

~~~text
node_modules
.next
package-lock.json
next-env.d.ts
coverage
playwright-report
test-results
~~~

**파일: `tsconfig.json`**

~~~json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts", ".next/dev/types/**/*.ts"],
  "exclude": ["node_modules"]
}
~~~

**파일: `next-env.d.ts`**

~~~typescript
/// <reference types="next" />
/// <reference types="next/image-types/global" />
~~~

**파일: `next.config.ts`**

~~~typescript
import type { NextConfig } from "next";

const config: NextConfig = { output: "standalone" };
export default config;
~~~

**파일: `eslint.config.mjs`**

~~~javascript
import { defineConfig, globalIgnores } from "eslint/config";
import nextVitals from "eslint-config-next/core-web-vitals";
import nextTypescript from "eslint-config-next/typescript";

export default defineConfig([
  ...nextVitals,
  ...nextTypescript,
  globalIgnores([".next/**", "next-env.d.ts", "test-results/**", "playwright-report/**"]),
]);
~~~

린트, 타입 검사와 빌드는 별도 명령으로 둡니다. 빌드가 성공했다고 코드 스타일과 모든 타입·테스트 검사가 끝났다고 가정하지 않습니다.

## 2. 개인별 DB와 환경변수 만들기

**파일: `.env.example`**

~~~dotenv
COMPOSE_PROJECT_NAME=ai-team-local
DB_PORT=55432
APP_PORT=3000
DATABASE_URL=postgresql://app:local_password@127.0.0.1:55432/app
DEV_API_KEY=local-development-only-change-me
~~~

이 값은 로컬 예제용입니다. 실제 서비스의 비밀번호나 API 키를 예제 파일에 넣지 않습니다. Next.js가 서버에서 읽는 환경변수와 브라우저로 노출하는 `NEXT_PUBLIC_` 변수는 구분해야 합니다. DB 접속 정보와 키에는 `NEXT_PUBLIC_`를 붙이지 않습니다. [환경변수 공식 문서](https://nextjs.org/docs/app/guides/environment-variables)를 참고할 수 있습니다.

**파일: `compose.yaml`**

~~~yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: local_password
      POSTGRES_DB: app
    ports:
      - "127.0.0.1:${DB_PORT:-55432}:5432"
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 2s
      timeout: 3s
      retries: 30

  migrate:
    profiles: ["app"]
    build:
      context: .
      target: tools
    command: ["npm", "run", "db:migrate"]
    environment:
      DATABASE_URL: postgresql://app:local_password@db:5432/app
    depends_on:
      db:
        condition: service_healthy

  app:
    profiles: ["app"]
    build:
      context: .
      target: runner
    environment:
      DATABASE_URL: postgresql://app:local_password@db:5432/app
      DEV_API_KEY: ${DEV_API_KEY:?Set DEV_API_KEY in .env}
      HOSTNAME: 0.0.0.0
      PORT: 3000
    ports:
      - "127.0.0.1:${APP_PORT:-3000}:3000"
    depends_on:
      migrate:
        condition: service_completed_successfully
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:3000/api/health').then(r=>{if(!r.ok)process.exit(1)}).catch(()=>process.exit(1))"]
      interval: 5s
      timeout: 5s
      retries: 20

volumes:
  db-data:
~~~

コンテナ内の `db` は Compose のサービス名です。ホスト側のアプリからは `127.0.0.1:55432`、コンテナ内からは `db:5432` に接続します。

起動順だけでは DB が接続可能とは限りません。ヘルスチェックとマイグレーション完了条件を使います。動作は [Compose 起動順の公式説明](https://docs.docker.com/compose/how-tos/startup-order/)で確認できます。

~~~bash
cp .env.example .env
docker compose up -d --wait db
~~~

같은 PC에서 여러 작업을 병렬로 진행할 때는 `.env`의 `COMPOSE_PROJECT_NAME`, `DB_PORT`, `DATABASE_URL`, `APP_PORT`를 작업별로 바꿉니다. 예를 들어 두 번째 작업은 프로젝트명 `ai-team-search`, DB 포트 `55433`, 앱 포트 `3001`을 사용합니다. 프로젝트명만 바꾸면 호스트 포트 충돌까지 해결되는 것은 아닙니다.

## 3. DB 변경을 코드로 관리한다

팀원이 각자 DB 콘솔에서 테이블을 만들면 새 팀원과 CI가 같은 상태를 재현할 수 없습니다. 스키마 변경은 순서가 있는 마이그레이션 파일로 남깁니다.

**파일: `migrations/202609070001_create-tasks.cjs`**

~~~javascript
exports.up = (pgm) => {
  pgm.sql(`
    CREATE TABLE tasks (
      id uuid PRIMARY KEY,
      title text NOT NULL CHECK (char_length(title) BETWEEN 1 AND 120),
      created_at timestamptz NOT NULL DEFAULT now()
    );
    CREATE INDEX tasks_created_at_id_idx ON tasks (created_at DESC, id DESC);
  `);
};

exports.down = false;
~~~

`down = false`는 이 예제의 자동 역방향 마이그레이션을 막습니다. 테이블을 지워 데이터까지 없애는 롤백을 기본값으로 제공하지 않기 위해서입니다. 오류가 있으면 새 마이그레이션으로 수정하고, 운영 데이터 복구는 백업 절차로 구분합니다.

~~~bash
npm run db:migrate
~~~

적용 이력은 마이그레이션 도구가 관리합니다. 이미 공유된 파일을 수정하지 않고 새로운 파일을 추가합니다. 두 사람이 동시에 마이그레이션을 만들었다면 병합 전에 순서와 상호 의존성을 확인합니다. [node-pg-migrate 시작 안내](https://salsita.github.io/node-pg-migrate/getting-started)와 [마이그레이션 정의](https://salsita.github.io/node-pg-migrate/migrations/)를 참고할 수 있습니다.

## 4. 최소 기능을 책임별로 구현한다

### DB 연결과 개발용 인증

**파일: `src/lib/db.ts`**

~~~typescript
import "server-only";
import { Pool } from "pg";

const state = globalThis as unknown as { appPool?: Pool };

export function getPool(): Pool {
  if (!state.appPool) {
    const connectionString = process.env.DATABASE_URL;
    if (!connectionString) throw new Error("DATABASE_URL is required");
    state.appPool = new Pool({
      connectionString,
      max: 5,
      connectionTimeoutMillis: 3000,
      statement_timeout: 5000,
    });
    state.appPool.on("error", () => console.error("database_pool_error"));
  }
  return state.appPool;
}
~~~

연결 풀을 재사용하고 요청마다 새 풀을 만들지 않습니다. 여러 앱 인스턴스를 띄우면 각 인스턴스의 최대 연결 수를 합산해 DB 한도를 계산해야 합니다. [node-postgres 연결 풀 설명](https://node-postgres.com/features/pooling)을 참고합니다.

**파일: `src/lib/auth.ts`**

~~~typescript
import "server-only";
import { timingSafeEqual } from "node:crypto";

export function isAuthorized(request: Request): boolean {
  const secret = process.env.DEV_API_KEY;
  if (!secret) return false;
  const actual = Buffer.from(request.headers.get("authorization") ?? "");
  const expected = Buffer.from(`Bearer ${secret}`);
  return actual.length === expected.length && timingSafeEqual(actual, expected);
}
~~~

이 공유 키는 개발용 API 호출 확인 수단입니다. 사용자 식별이나 조직별 권한을 제공하지 않습니다. 키가 없으면 허용하지 않는 동작을 기본값으로 두고, 공개 서비스 전에는 조직의 로그인·역할·테넌트 권한 모델로 교체합니다.

### 입력 계약과 저장 계층

**파일: `src/features/tasks/schema.ts`**

~~~typescript
import { z } from "zod";

export const createTaskSchema = z.object({
  title: z.string().trim().min(1).max(120),
}).strict();
~~~

**파일: `src/features/tasks/repository.ts`**

~~~typescript
import "server-only";
import { randomUUID } from "node:crypto";
import { getPool } from "@/lib/db";

export type Task = { id: string; title: string; created_at: Date };

export async function createTask(title: string): Promise<Task> {
  const result = await getPool().query<Task>(
    "INSERT INTO tasks (id, title) VALUES ($1, $2) RETURNING id, title, created_at",
    [randomUUID(), title],
  );
  return result.rows[0];
}

export async function listTasks(): Promise<Task[]> {
  const result = await getPool().query<Task>(
    "SELECT id, title, created_at FROM tasks ORDER BY created_at DESC, id DESC LIMIT 20",
  );
  return result.rows;
}
~~~

SQL에 입력 문자열을 이어 붙이지 않고 매개변수로 전달합니다. [node-postgres 매개변수 쿼리](https://node-postgres.com/features/queries)에서 이 방식을 설명합니다. 목록은 최신 20개로 제한하며, 다음 단계에서 페이지네이션을 별도 계약으로 추가합니다.

### API와 시작 화면

**파일: `src/app/api/tasks/route.ts`**

~~~typescript
import { isAuthorized } from "@/lib/auth";
import { createTaskSchema } from "@/features/tasks/schema";
import { createTask, listTasks } from "@/features/tasks/repository";

export const runtime = "nodejs";
export const dynamic = "force-dynamic";

export async function GET(request: Request) {
  if (!isAuthorized(request)) {
    return Response.json({ error: "unauthorized" }, { status: 401 });
  }
  try {
    return Response.json(await listTasks());
  } catch {
    console.error("tasks_list_failed");
    return Response.json({ error: "unavailable" }, { status: 503 });
  }
}

export async function POST(request: Request) {
  if (!isAuthorized(request)) {
    return Response.json({ error: "unauthorized" }, { status: 401 });
  }
  let body: unknown;
  try {
    body = await request.json();
  } catch {
    return Response.json({ error: "invalid_json" }, { status: 400 });
  }
  const parsed = createTaskSchema.safeParse(body);
  if (!parsed.success) {
    return Response.json({ error: "invalid_input" }, { status: 400 });
  }
  try {
    return Response.json(await createTask(parsed.data.title), { status: 201 });
  } catch {
    console.error("tasks_create_failed");
    return Response.json({ error: "unavailable" }, { status: 503 });
  }
}
~~~

**파일: `src/app/api/health/route.ts`**

~~~typescript
import { getPool } from "@/lib/db";

export const runtime = "nodejs";
export const dynamic = "force-dynamic";

export async function GET() {
  try {
    await getPool().query("SELECT id FROM tasks LIMIT 0");
    return Response.json({ status: "ok" });
  } catch {
    return Response.json({ status: "unavailable" }, { status: 503 });
  }
}
~~~

헬스 체크는 프로세스의 생존뿐 아니라 DB 연결과 테이블 적용 여부까지 확인합니다. 실제 운영에서는 프로세스 생존을 보는 liveness와 요청 처리 준비 상태를 보는 readiness를 분리할 수 있습니다.

**파일: `src/app/layout.tsx`**

~~~tsx
import type { ReactNode } from "react";

export default function RootLayout({ children }: { children: ReactNode }) {
  return <html lang="ko"><body>{children}</body></html>;
}
~~~

**파일: `src/app/page.tsx`**

~~~tsx
export default function Home() {
  return (
    <main>
      <h1>AI Team Starter</h1>
      <p>팀 개발용 API 프로젝트입니다.</p>
      <a href="/api/health">서비스 상태 확인</a>
    </main>
  );
}
~~~

route 파일의 HTTP 메서드와 응답 처리 방식은 [Next.js Route Handlers](https://nextjs.org/docs/app/api-reference/file-conventions/route)를 기준으로 합니다.

~~~bash
npm run dev
~~~

다른 터미널에서 확인합니다. 아래 키는 `.env.example`의 로컬 값과 같습니다. 키를 변경했다면 요청도 같은 값으로 바꿉니다.

~~~bash
curl --fail-with-body http://127.0.0.1:3000/api/health
curl --fail-with-body http://127.0.0.1:3000/api/tasks \
  -H 'Authorization: Bearer local-development-only-change-me' \
  -H 'Content-Type: application/json' \
  -d '{"title":"첫 번째 팀 작업"}'
curl --fail-with-body http://127.0.0.1:3000/api/tasks \
  -H 'Authorization: Bearer local-development-only-change-me'
~~~

정상이라면 상태 응답, 생성한 작업, 작업 배열을 순서대로 받습니다. 키 없이 작업 API를 호출하면 `401`, 빈 제목이면 `400`을 반환합니다.

## 5. AI가 작성한 코드도 같은 기준으로 검사한다

### 단위 테스트: 입력 계약을 고정한다

**파일: `vitest.config.ts`**

~~~typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: { environment: "node", include: ["tests/unit/**/*.test.ts"] },
});
~~~

**파일: `tests/unit/task-schema.test.ts`**

~~~typescript
import { describe, expect, it } from "vitest";
import { createTaskSchema } from "../../src/features/tasks/schema";

describe("작업 생성 계약", () => {
  it("앞뒤 공백을 제거한다", () => {
    expect(createTaskSchema.parse({ title: "  작업  " })).toEqual({ title: "작업" });
  });

  it.each(["", "   ", "a".repeat(121)])("잘못된 제목을 거절한다: %s", (title) => {
    expect(createTaskSchema.safeParse({ title }).success).toBe(false);
  });

  it("계약에 없는 필드를 거절한다", () => {
    expect(createTaskSchema.safeParse({ title: "작업", admin: true }).success).toBe(false);
  });
});
~~~

단위 테스트는 실행 환경이 단순한 입력 계약부터 시작합니다. 비동기 서버 컴포넌트를 무리하게 단위 테스트 도구에 맞추지 않고 API나 브라우저 테스트로 확인할 수 있습니다. [Next.js Vitest 가이드](https://nextjs.org/docs/app/guides/testing/vitest)를 참고합니다.

### API 테스트: 실제 서버와 DB를 연결한다

**파일: `playwright.config.ts`**

~~~typescript
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "tests/api",
  workers: 1,
  use: { baseURL: "http://127.0.0.1:3100" },
  webServer: {
    command: "npm run start -- --port 3100",
    url: "http://127.0.0.1:3100/api/health",
    reuseExistingServer: false,
    timeout: 60000,
  },
});
~~~

**파일: `tests/api/tasks.spec.ts`**

~~~typescript
import { randomUUID } from "node:crypto";
import { expect, test } from "@playwright/test";

test("인증 없는 조회는 거절한다", async ({ request }) => {
  const response = await request.get("/api/tasks");
  expect(response.status()).toBe(401);
});

test("빈 제목과 잘못된 JSON은 저장하지 않는다", async ({ request }) => {
  const headers = { Authorization: `Bearer ${process.env.DEV_API_KEY}` };
  const empty = await request.post("/api/tasks", { headers, data: { title: " " } });
  expect(empty.status()).toBe(400);
  const malformed = await request.post("/api/tasks", {
    headers: { ...headers, "Content-Type": "application/json" },
    data: "{broken",
  });
  expect(malformed.status()).toBe(400);
});

test("생성한 작업이 실제 조회 결과에 있다", async ({ request }) => {
  const headers = { Authorization: `Bearer ${process.env.DEV_API_KEY}` };
  const title = `integration-${randomUUID()}`;
  const created = await request.post("/api/tasks", { headers, data: { title } });
  expect(created.status()).toBe(201);
  const task = await created.json();
  expect(task).toMatchObject({ title });
  const listed = await request.get("/api/tasks", { headers });
  expect(listed.status()).toBe(200);
  expect(await listed.json()).toEqual(expect.arrayContaining([
    expect.objectContaining({ id: task.id, title }),
  ]));
});
~~~

이번 테스트는 Playwright의 HTTP 요청 기능만 사용하므로 브라우저 바이너리를 설치하지 않습니다. UI 기능을 추가하면 브라우저 설치와 사용자 흐름 테스트를 추가합니다. 서버 시작을 포함한 설정은 [Next.js Playwright 가이드](https://nextjs.org/docs/app/guides/testing/playwright)를 참고할 수 있습니다.

이 테스트는 작업 데이터를 실제로 추가합니다. 운영 DB를 지정하지 않고 개인의 개발 DB나 CI 전용 DB에서 실행합니다. 운영 데이터를 지우는 공통 초기화 명령을 만들지 않습니다.

~~~bash
npm run format
npm run check
npm run test:api
~~~

`check`는 DB를 쓰지 않는 검사와 빌드를, `test:api`는 마이그레이션된 실제 DB가 필요한 검사를 담당합니다. 테스트 실행 중에는 포트 `3100`을 비워둡니다.

## 6. 공통 규칙을 AGENTS.md로 남긴다

AI에게 매번 긴 설명을 복사하기보다 저장소의 기준 문서를 읽게 합니다. [AGENTS.md](https://agents.md/)는 코딩 에이전트에 프로젝트 맥락과 명령을 전달하기 위한 공개 형식입니다. 도구마다 자동 탐색과 우선순위가 다를 수 있으므로 사용하는 도구에서 적용 여부를 확인합니다.

**파일: `AGENTS.md`**

~~~markdown
# AI Team Starter 작업 규칙

## 먼저 읽기
- README.md: 실행 방법과 검증 명령
- docs/architecture.md: 계층과 의존 방향
- docs/tasks/: 이번 작업의 요구사항과 완료 기준
- docs/adr/: 기존 설계 결정

## 구현 원칙
- route는 인증·입출력·상태 코드를 담당한다.
- 입력 계약은 src/features/<기능>/schema.ts에 둔다.
- SQL은 repository.ts에 두고 매개변수 쿼리를 사용한다.
- 브라우저 코드에서 DB와 서버 비밀값에 접근하지 않는다.
- 이미 공유된 마이그레이션을 고치지 않고 새 파일을 추가한다.
- 요청 범위를 벗어난 의존성·아키텍처 변경은 별도 제안으로 남긴다.

## 작업 방식
- 시작할 때 git status와 관련 코드·테스트를 확인한다.
- 다른 사람이 만든 변경을 덮어쓰지 않는다.
- 작업 범위와 충돌 가능 파일을 확인한 뒤 수정한다.
- 동작이 바뀌면 그 계약을 검증하는 테스트를 함께 수정한다.
- 테스트를 삭제하거나 실패를 무시하는 옵션으로 통과시키지 않는다.

## 검증
- npm run check
- DB 관련 변경이면 npm run db:migrate 후 npm run test:api
- 실행하지 못한 검사는 완료 보고에 명시한다.

## 권한 경계
- .env, 운영 데이터와 실제 토큰을 답변·로그·커밋에 넣지 않는다.
- 테스트에는 개인 개발 DB나 CI 전용 DB만 사용한다.
- main 직접 푸시, 운영 배포와 운영 마이그레이션은 작업자의 별도 승인을 따른다.

## 완료 보고
- 변경한 사용자 동작과 관련 파일
- 실행한 검증과 결과
- 남은 제약, 데이터 변경과 배포 시 주의점
~~~

이 파일은 **지시 문서이며 접근 제어 장치가 아닙니다.** 운영 비밀값 접근이나 병합 권한은 실제 계정·도구 권한과 저장소 보호 규칙으로 제한합니다. AI의 답변에 “테스트 통과”라고 적혀 있다는 것과 CI가 통과했다는 것도 구분해야 합니다.

### 사람이 읽는 문서도 같은 저장소에 둔다

**파일: `README.md`**

~~~markdown
# AI Team Starter

Node.js 24, npm, Docker Compose v2가 필요합니다.

## 처음 실행
1. .nvmrc에 맞는 Node.js를 사용합니다.
2. npm ci
3. cp .env.example .env
4. docker compose up -d --wait db
5. npm run db:migrate
6. npm run dev

## 검증
- npm run check
- npm run test:api: 실행 가능한 DB가 필요하며 테스트 데이터를 추가합니다.

## 컨테이너 확인
- docker compose --profile app up --build -d --wait app
- 중지: docker compose --profile app down
- 볼륨은 삭제하지 않으며 DB 데이터가 유지됩니다.

## 문서
- docs/architecture.md
- docs/adr/0001-modular-monolith.md
- docs/runbook.md

로컬 개발용 공유 키를 사용합니다. 공개 서비스용 사용자 인증은 별도로 구현합니다.
~~~

**파일: `docs/architecture.md`**

~~~markdown
# 아키텍처

- src/app: HTTP와 페이지 경계
- src/features/tasks/schema.ts: 외부 입력 계약
- src/features/tasks/repository.ts: PostgreSQL 접근
- src/lib: 서버 공통 기능
- migrations: 공유된 DB 변경 이력

route에서 repository를 호출하며 repository는 route를 참조하지 않는다.
업무 규칙이 복잡해지면 service 계층을 추가한다.
외부 API와 AI 모델 호출을 추가하면 adapter로 분리하고 timeout·재시도·비용 한도를 정한다.
현재 저장소에는 서비스 내부의 AI 모델 호출 기능이 없다.

API 계약:
- GET /api/health: 준비 완료 200, DB 또는 스키마 문제 503
- GET /api/tasks: 인증 후 최신 20개, 인증 실패 401
- POST /api/tasks: title 1~120자, 성공 201, 입력 오류 400, 인증 실패 401
- 저장·조회 실패: 503, 내부 오류 본문과 DB 접속 정보는 외부에 반환하지 않음
~~~

**파일: `docs/adr/0001-modular-monolith.md`**

~~~markdown
# ADR 0001: 단일 애플리케이션으로 시작

상태: 채택

배경: 작은 팀이 AI를 사용해 함께 개발하며 초기 운영 비용을 줄여야 한다.
결정: Next.js 단일 앱과 PostgreSQL을 사용하고 기능별 디렉터리로 책임을 나눈다.
대안: 여러 서비스로 나누면 독립 배포가 가능하지만 통신·운영·통합 테스트가 늘어난다.
비용: 초기에 배포 단위가 공유되므로 기능별 릴리스 독립성이 낮다.
재검토 조건: 특정 기능의 부하·보안·팀 소유권 때문에 독립 배포가 필요해질 때.
~~~

ADR은 설계 결정의 이유를 남기는 문서입니다. AI나 새 팀원이 “더 좋아 보이는 구조”로 같은 결정을 반복해서 뒤집는 일을 줄입니다. 위 ADR의 선택은 이 예제를 위한 설계 판단이며 모든 팀의 정답은 아닙니다.

## 7. 사람과 AI의 병렬 작업을 분리한다

사람별로 저장소를 clone하고 **한 작업에 한 브랜치**를 사용합니다. 한 사람이 같은 PC에서 두 작업을 진행할 때는 worktree로 작업 디렉터리까지 나눌 수 있습니다.

~~~bash
git switch -c feat/task-pagination

# 초기 main 커밋이 존재할 때, 별도 작업 디렉터리 추가
git worktree add ../ai-team-search -b feat/task-search main
~~~

worktree는 파일과 브랜치를 분리하지만 외부 DB와 포트까지 분리하지는 않습니다. 각 디렉터리에서 `npm ci`와 `.env` 생성을 수행하고 앞서 설명한 프로젝트명·포트를 각각 지정합니다. [git worktree 공식 문서](https://git-scm.com/docs/git-worktree)를 참고합니다.

### 작업 요청을 검증 가능한 문서로 만든다

**파일: `docs/tasks/001-pagination.md`**

~~~markdown
# 작업 001: 작업 목록 페이지네이션

목표: 최신 20개 이후의 작업을 중복 없이 조회한다.

범위:
- tasks 목록 repository와 GET API
- 관련 계약 문서와 API 테스트

제외:
- 인증 방식 변경
- 신규 라이브러리 추가
- 기존 마이그레이션 수정

완료 기준:
- created_at과 id를 함께 사용하는 커서를 정의한다.
- 커서가 없으면 첫 페이지를 반환한다.
- 잘못된 커서는 400을 반환한다.
- 25개 이상의 데이터를 넣고 다음 페이지에 중복이 없음을 검증한다.
- 기존 목록 응답과 달라지는 형식을 문서에 명시한다.

검증: npm run check, npm run test:api
~~~

이 문서는 **다음 작업의 예시**이며 현재 코드는 첫 20개만 반환합니다. AI에게 다음처럼 요청할 수 있습니다.

~~~text
AGENTS.md, docs/architecture.md, docs/tasks/001-pagination.md를 읽어줘.
관련 구현과 테스트를 확인하고 변경할 파일과 호환성 영향을 정리한 뒤 구현해줘.
기존 변경을 보존하고 이번 작업의 범위 안에서 수정해줘.
완료 기준을 테스트로 확인하고 실행한 명령과 결과를 보고해줘.
~~~

두 작업이 같은 API 계약이나 잠금 파일을 바꾸면 작업자를 분리해도 충돌할 수 있습니다. **API 계약·마이그레이션·의존성 변경의 병합 순서**를 먼저 합의합니다. 예를 들어 한 사람이 응답 계약을 먼저 확정하고, 다른 사람이 그 계약을 기준으로 UI를 작성합니다.

## 8. GitHub Actions를 실제 병합 조건으로 연결한다

코드 검사뿐 아니라 빈 DB에서 마이그레이션과 API 테스트가 실행되어야 새 팀원이 프로젝트를 재현할 수 있습니다.

**파일: `.github/workflows/ci.yml`**

{% raw %}
~~~yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  verify:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: ci_password
          POSTGRES_DB: app_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U app -d app_test"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    env:
      DATABASE_URL: postgresql://app:ci_password@127.0.0.1:5432/app_test
      DEV_API_KEY: ci-only-development-key
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
      - run: npm ci
      - run: npm run check
      - run: npm run db:migrate
      - run: npm run test:api
      - run: docker build --target runner -t ai-team-starter:${{ github.sha }} .
~~~
{% endraw %}

이 예제의 CI 값은 해당 실행에서 생성하는 일회성 DB용입니다. 운영 비밀값을 PR 검증에 전달하지 않습니다. 외부 PR 코드를 실행하면서 높은 권한을 주는 `pull_request_target` 구성도 사용하지 않습니다.

예제는 읽기 쉬운 메이저 태그를 사용했습니다. 운영 저장소에서는 공식 action 저장소에서 검증한 **전체 커밋 SHA**로 고정하고 업데이트 PR로 관리합니다. 태그는 이동할 수 있다는 차이가 있습니다. [GitHub Actions 보안 안내](https://docs.github.com/en/actions/reference/security/secure-use)를 참고합니다.

### 리뷰와 보호 규칙

**파일: `.github/pull_request_template.md`**

~~~markdown
## 문제와 변경된 동작

## 확인 방법과 실행 결과
- [ ] npm run check
- [ ] DB 관련 변경이면 마이그레이션과 API 테스트

## 데이터·호환성·운영 영향
- 마이그레이션:
- 환경변수:
- API 호환성:
- 배포 실패 시 대응:

## AI 사용 시 작성자 확인
- [ ] 생성한 코드를 이해하고 요구사항과 비교했다.
- [ ] 테스트가 실제 실패 조건을 검증하는지 확인했다.
- [ ] 비밀값과 요청 범위 밖 변경이 없는지 확인했다.
~~~

`CODEOWNERS`를 사용할 경우 아래 계정명을 실제로 **저장소 write 권한이 있는 팀원**으로 바꾼 뒤 파일을 활성화합니다. 팀 계정은 조직 저장소에서 가시성과 권한을 확인해야 합니다.

~~~text
# .github/CODEOWNERS의 예시 — 계정명 교체 필요
* @your-maintainer
/.github/ @your-platform-reviewer
/migrations/ @your-db-reviewer
/AGENTS.md @your-maintainer
~~~

파일만 추가하면 리뷰가 강제되는 것은 아닙니다. GitHub의 `main` 보호 규칙 또는 ruleset에 다음을 설정합니다.

1. PR을 통해서만 병합합니다.
2. 작성자 외 최소 1명의 승인을 요구합니다.
3. CODEOWNERS를 사용한다면 코드 소유자의 승인을 요구합니다.
4. CI를 한 번 실행한 뒤 실제 표시되는 `verify` 체크를 필수 검사로 선택합니다.
5. 새 커밋이 추가되면 이전 승인을 무효화하고, 미해결 대화를 해결하도록 합니다.
6. 최신 main과의 검증을 요구하거나 팀 환경에 맞는 merge queue를 구성합니다.
7. 강제 푸시와 브랜치 삭제를 막고 관리자 우회 범위를 최소화합니다.

사용 가능한 규칙은 저장소 공개 여부와 요금제에 따라 다를 수 있습니다. [브랜치 보호](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)와 [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) 문서를 확인합니다.

## 9. 같은 결과물을 컨테이너로 실행한다

**파일: `.dockerignore`**

~~~text
node_modules
.next
.git
.env*
coverage
playwright-report
test-results
~~~

**파일: `Dockerfile`**

~~~dockerfile
FROM node:24-bookworm-slim AS tools
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .

FROM tools AS build
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:24-bookworm-slim AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV HOSTNAME=0.0.0.0
ENV PORT=3000
COPY --from=build --chown=node:node /app/.next/standalone ./
COPY --from=build --chown=node:node /app/.next/static ./.next/static
USER node
EXPOSE 3000
CMD ["node", "server.js"]
~~~

`standalone` 출력을 사용해 실행에 필요한 파일을 모읍니다. 이 예제에는 `public` 디렉터리가 없으며, 정적 파일을 추가했다면 런타임 이미지에 `public`도 복사해야 합니다. [Next.js standalone 설명](https://nextjs.org/docs/app/api-reference/config/next-config-js/output)을 참고합니다.

개발 서버가 같은 포트를 사용 중이면 먼저 종료하고 다음을 실행합니다.

~~~bash
docker compose --profile app up --build -d --wait app
curl --fail-with-body http://127.0.0.1:3000/api/health
docker compose --profile app logs --tail=50 app
~~~

DB 준비, 마이그레이션, 앱 실행의 순서로 동작합니다. 이 Compose 파일은 개발·컨테이너 검증용이며 포트를 로컬 주소에만 공개합니다. 공유 서버에 배포할 때는 개발 비밀번호와 공유 키를 그대로 사용하지 않습니다.

### 운영 배포는 검증된 이미지와 데이터 변경을 함께 다룬다

실제 배포의 기본 순서는 다음과 같이 정할 수 있습니다.

~~~text
PR 검사와 사람 리뷰
  → main 병합 후 동일 검사
  → 커밋 SHA 태그로 이미지 생성·레지스트리 저장
  → 스테이징에서 마이그레이션·헬스 체크·핵심 기능 확인
  → 배포 담당자의 승인
  → 운영 백업 확인과 호환되는 마이그레이션
  → 같은 이미지 digest를 운영에 배포
  → 상태 확인, 오류율·지연 시간 관찰
~~~

배포 환경마다 클라우드 권한과 실행 플랫폼이 다르므로 이 글의 CI는 이미지를 외부에 푸시하거나 운영에 자동 배포하지 않습니다. 운영 배포를 추가할 때는 인증·권한, HTTPS, 비밀 관리, DB 백업·복구, 요청 크기 제한, 접근 로그와 경보를 해당 환경에 맞게 연결합니다.

앱 이미지를 이전 버전으로 되돌려도 이미 바꾼 DB 스키마가 자동으로 돌아가지는 않습니다. 먼저 새 열을 추가하고 구버전과 신버전이 함께 동작하게 만든 뒤, 별도 배포에서 이전 열을 제거하는 식으로 변경을 나눕니다.

**파일: `docs/runbook.md`**

~~~markdown
# 실행·장애 대응

## 담당자
- 서비스 담당: 저장소 관리자가 실제 이름으로 지정
- DB와 복구 담당: 실제 이름으로 지정

## 로컬 상태 확인
- docker compose ps
- curl --fail-with-body http://127.0.0.1:3000/api/health
- docker compose --profile app logs --tail=100 app

## 503 응답
1. DB 서비스가 실행 중인지 확인한다.
2. DATABASE_URL의 호스트·포트가 실행 위치와 일치하는지 확인한다.
3. 마이그레이션 적용 실패를 확인한다.
4. 키·DB URL·사용자 입력 전체를 공용 로그에 복사하지 않는다.

## 배포 실패
- 배포 커밋과 이전 이미지 digest를 기록한다.
- DB 변경과 이전 앱의 호환성을 확인한 뒤 앱을 되돌린다.
- 데이터 손실이 의심되면 추가 쓰기를 제한하고 복구 담당자에게 전달한다.
- 백업 복구는 별도 DB에서 검증한 뒤 전환한다.

## 개발 데이터 보존
- docker compose --profile app down은 볼륨을 남긴다.
- 볼륨 삭제는 별도 작업이며 일반적인 중지·재시작 절차에 포함하지 않는다.
~~~

## 10. 첫 병합과 새 팀원 온보딩

모든 파일을 만든 뒤 포맷과 검증을 실행합니다. `.env`가 추적되지 않는지도 확인합니다.

~~~bash
npm run format
npm run check
npm run db:migrate
npm run test:api
git status --short
git check-ignore .env
git diff --check
~~~

첫 담당자는 결과를 확인한 뒤 초기 커밋을 생성하고 팀 저장소에 올립니다. 원격 저장소 URL은 실제 팀 저장소로 바꿉니다.

~~~bash
git add .
git commit -m "chore: initialize team development baseline"
git remote add origin https://github.com/YOUR-ORG/ai-team-starter.git
git push -u origin main
~~~

이 초기화 직후 CI 실행 결과를 확인하고 앞서 설명한 main 보호 규칙을 활성화합니다. 그다음부터는 작업 브랜치와 PR을 사용합니다. 새 팀원은 저장소를 clone한 뒤 README의 명령으로 시작할 수 있어야 합니다.

최종 완료 기준은 다음과 같습니다.

- 새 clone에서 `npm ci`로 의존성을 재현할 수 있습니다.
- 빈 DB에서 마이그레이션을 적용하고 API를 실행할 수 있습니다.
- 잘못된 입력과 인증 없는 요청이 테스트에서 거절됩니다.
- CI가 실패하거나 필수 리뷰가 없으면 main에 병합할 수 없습니다.
- AI는 공통 규칙과 작업 문서를 읽고 같은 검증 명령을 사용합니다.
- 담당자가 바뀌어도 설계 이유와 실행·복구 절차를 저장소에서 찾을 수 있습니다.

이후 AI에게 기능을 맡길 때는 구현할 내용뿐 아니라 **변경 가능한 범위, 유지해야 할 계약, 검증할 결과**를 함께 제공합니다. 팀이 유지보수하는 대상은 AI와 나눈 대화가 아니라, 검증 가능한 코드와 그 결정을 설명하는 저장소입니다.
