



从0开始构建类似 SQL Chat 的项目需要理解其核心架构和技术栈。让我为你提供一个完整的从头开发指南：

## 核心架构分析

SQL Chat 的核心功能：
1. **自然语言转 SQL** - 使用 OpenAI API 将用户提问转换为 SQL
2. **数据库连接管理** - 支持多种数据库的连接和查询
3. **聊天界面** - 类似 ChatGPT 的对话式界面
4. **会话管理** - 保存历史对话记录

## 技术栈选择

| 分类 | 技术 | 版本 |
|------|------|------|
| 框架 | Next.js | 13.x |
| UI | React | 18.x |
| 样式 | Tailwind CSS | 3.x |
| ORM | Prisma | 4.x |
| 认证 | NextAuth | 4.x |
| 数据库 | PostgreSQL | - |

## 从零开始构建

### 1. 初始化 Next.js 项目

```bash
npx create-next-app@13.5.6 . --typescript
```

### 2. 安装核心依赖

```bash
pnpm install @prisma/client @next-auth/prisma-adapter next-auth openai
pnpm install -D prisma tailwindcss @tailwindcss/typography
```

### 3. 配置 Tailwind CSS

创建 `tailwind.config.js`：

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [require('@tailwindcss/typography')],
}
```

更新 `src/styles/globals.css`：

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 4. 配置 Prisma

初始化 Prisma：

```bash
npx prisma init
```

更新 `prisma/schema.prisma`：

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime?
  image         String?
  accounts      Account[]
  sessions      Session[]
  connections   Connection[]
  conversations Conversation[]
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

model Connection {
  id          String     @id @default(cuid())
  userId      String
  name        String
  type        String     // mysql, postgresql, mssql
  host        String
  port        String
  database    String
  username    String
  password    String
  createdAt   DateTime   @default(now())
  user        User       @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model Conversation {
  id          String     @id @default(cuid())
  userId      String
  title       String
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
  messages    Message[]
  user        User       @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model Message {
  id             String     @id @default(cuid())
  conversationId String
  role           String     // user, assistant
  content        String
  sql            String?
  createdAt      DateTime   @default(now())
  conversation   Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)
}
```

### 5. 配置环境变量

创建 `.env` 文件：

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/sqlchat?schema=sqlchat"
OPENAI_API_KEY=your-openai-api-key
NEXTAUTH_SECRET=your-secret-key
NEXTAUTH_URL=http://localhost:3000
```

### 6. 创建数据库迁移

```bash
npx prisma migrate dev --name init
```

### 7. 创建 OpenAI 服务

创建 `src/services/openai.ts`：

```typescript
import { OpenAI } from 'openai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  baseURL: process.env.OPENAI_API_ENDPOINT || 'https://api.openai.com/v1',
});

export async function generateSQL(prompt: string, databaseSchema: string): Promise<string> {
  const response = await openai.chat.completions.create({
    model: 'gpt-4',
    messages: [
      {
        role: 'system',
        content: `You are a SQL expert. Given a natural language query and database schema, generate valid SQL.\n\nDatabase Schema:\n${databaseSchema}`,
      },
      {
        role: 'user',
        content: prompt,
      },
    ],
    temperature: 0,
  });

  return response.choices[0].message.content || '';
}
```

### 8. 创建数据库连接服务

创建 `src/services/database.ts`：

```typescript
import { Connection } from '@prisma/client';
import mysql from 'mysql2/promise';
import pg from 'pg';
import sql from 'mssql';

export async function executeQuery(connection: Connection, sqlQuery: string): Promise<any[]> {
  switch (connection.type) {
    case 'mysql': {
      const db = await mysql.createConnection({
        host: connection.host,
        port: parseInt(connection.port),
        user: connection.username,
        password: connection.password,
        database: connection.database,
      });
      const [rows] = await db.query(sqlQuery);
      await db.end();
      return rows as any[];
    }
    case 'postgresql': {
      const client = new pg.Client({
        host: connection.host,
        port: parseInt(connection.port),
        user: connection.username,
        password: connection.password,
        database: connection.database,
      });
      await client.connect();
      const result = await client.query(sqlQuery);
      await client.end();
      return result.rows;
    }
    case 'mssql': {
      const pool = await sql.connect({
        server: connection.host,
        port: parseInt(connection.port),
        user: connection.username,
        password: connection.password,
        database: connection.database,
        options: { encrypt: true },
      });
      const result = await pool.request().query(sqlQuery);
      await pool.close();
      return result.recordset;
    }
    default:
      throw new Error('Unsupported database type');
  }
}

export async function getDatabaseSchema(connection: Connection): Promise<string> {
  // 实现获取数据库表结构的逻辑
  // 根据不同数据库类型查询 INFORMATION_SCHEMA 或类似系统表
  return '';
}
```

### 9. 创建 API 路由

创建 `src/pages/api/chat.ts`：

```typescript
import { NextApiRequest, NextApiResponse } from 'next';
import { getServerSession } from 'next-auth/next';
import { authOptions } from './auth/[...nextauth]';
import { PrismaClient } from '@prisma/client';
import { generateSQL } from '../../services/openai';
import { executeQuery, getDatabaseSchema } from '../../services/database';

const prisma = new PrismaClient();

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const session = await getServerSession(req, res, authOptions);
  
  if (!session?.user) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  const { conversationId, message, connectionId } = req.body;

  // 获取连接信息
  const connection = await prisma.connection.findUnique({
    where: { id: connectionId },
  });

  if (!connection) {
    return res.status(404).json({ error: 'Connection not found' });
  }

  // 获取数据库 schema
  const schema = await getDatabaseSchema(connection);

  // 生成 SQL
  const sql = await generateSQL(message, schema);

  // 执行 SQL
  const results = await executeQuery(connection, sql);

  // 保存消息到数据库
  const conversation = await prisma.conversation.upsert({
    where: { id: conversationId || '' },
    update: { updatedAt: new Date() },
    create: {
      userId: session.user.id!,
      title: message.substring(0, 50),
    },
  });

  await prisma.message.createMany({
    data: [
      { conversationId: conversation.id, role: 'user', content: message },
      { conversationId: conversation.id, role: 'assistant', content: JSON.stringify(results), sql },
    ],
  });

  res.json({ sql, results });
}
```

### 10. 创建 NextAuth 配置

创建 `src/pages/api/auth/[...nextauth].ts`：

```typescript
import NextAuth from 'next-auth';
import GoogleProvider from 'next-auth/providers/google';
import { PrismaAdapter } from '@next-auth/prisma-adapter';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export const authOptions = {
  adapter: PrismaAdapter(prisma),
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  session: {
    strategy: 'jwt',
  },
};

export default NextAuth(authOptions);
```

### 11. 创建聊天界面组件

创建 `src/components/ChatInterface.tsx`：

```tsx
import React, { useState } from 'react';
import { Send, Database, MessageSquare } from 'lucide-react';

export default function ChatInterface() {
  const [message, setMessage] = useState('');
  const [messages, setMessages] = useState<{ role: string; content: string; sql?: string }[]>([]);
  const [isLoading, setIsLoading] = useState(false);

  const handleSend = async () => {
    if (!message.trim()) return;

    setMessages(prev => [...prev, { role: 'user', content: message }]);
    setIsLoading(true);
    setMessage('');

    // 调用 API
    const response = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message,
        connectionId: 'your-connection-id', // 从状态中获取
      }),
    });

    const { sql, results } = await response.json();
    
    setMessages(prev => [...prev, { 
      role: 'assistant', 
      content: JSON.stringify(results, null, 2),
      sql 
    }]);
    setIsLoading(false);
  };

  return (
    <div className="flex flex-col h-screen bg-gray-50">
      {/* 头部 */}
      <header className="bg-white border-b border-gray-200 px-4 py-3 flex items-center gap-3">
        <Database className="w-6 h-6 text-blue-500" />
        <h1 className="text-lg font-semibold">SQL Chat</h1>
      </header>

      {/* 消息列表 */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((msg, index) => (
          <div key={index} className={`flex gap-3 ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
            <div className={`max-w-[70%] p-3 rounded-lg ${msg.role === 'user' ? 'bg-blue-500 text-white' : 'bg-white border border-gray-200'}`}>
              <p className="text-sm">{msg.content}</p>
              {msg.sql && (
                <pre className="mt-2 text-xs bg-gray-100 p-2 rounded text-gray-600">
                  {msg.sql}
                </pre>
              )}
            </div>
          </div>
        ))}
        {isLoading && (
          <div className="flex justify-center">
            <div className="w-6 h-6 border-2 border-blue-500 border-t-transparent rounded-full animate-spin" />
          </div>
        )}
      </div>

      {/* 输入框 */}
      <footer className="bg-white border-t border-gray-200 px-4 py-3">
        <div className="flex gap-3">
          <input
            type="text"
            value={message}
            onChange={(e) => setMessage(e.target.value)}
            onKeyPress={(e) => e.key === 'Enter' && handleSend()}
            placeholder="Ask me about your database..."
            className="flex-1 px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          <button
            onClick={handleSend}
            disabled={isLoading}
            className="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            <Send className="w-5 h-5" />
          </button>
        </div>
      </footer>
    </div>
  );
}
```

### 12. 更新首页

更新 `src/pages/index.tsx`：

```tsx
import { getServerSession } from 'next-auth/next';
import { authOptions } from './api/auth/[...nextauth]';
import ChatInterface from '../components/ChatInterface';
import SignIn from '../components/SignIn';

export default async function Home({ session }: { session: any }) {
  if (!session) {
    return <SignIn />;
  }

  return <ChatInterface />;
}

export async function getServerSideProps(context: any) {
  const session = await getServerSession(context.req, context.res, authOptions);
  return { props: { session } };
}
```

### 13. 创建登录组件

创建 `src/components/SignIn.tsx`：

```tsx
import { signIn } from 'next-auth/react';
import { Google, MessageSquare } from 'lucide-react';

export default function SignIn() {
  return (
    <div className="min-h-screen flex items-center justify-center bg-gradient-to-br from-blue-50 to-indigo-100">
      <div className="bg-white rounded-2xl shadow-xl p-8 max-w-md w-full mx-4">
        <div className="text-center mb-8">
          <div className="w-16 h-16 bg-blue-500 rounded-full flex items-center justify-center mx-auto mb-4">
            <MessageSquare className="w-8 h-8 text-white" />
          </div>
          <h1 className="text-2xl font-bold text-gray-800">SQL Chat</h1>
          <p className="text-gray-500 mt-2">Chat with your database using natural language</p>
        </div>

        <button
          onClick={() => signIn('google')}
          className="w-full flex items-center justify-center gap-3 px-4 py-3 bg-white border border-gray-300 rounded-lg hover:bg-gray-50 transition-colors"
        >
          <Google className="w-5 h-5" />
          <span>Sign in with Google</span>
        </button>
      </div>
    </div>
  );
}
```

### 14. 创建连接管理组件

创建 `src/components/ConnectionSidebar.tsx`：

```tsx
import React, { useState } from 'react';
import { Plus, Database, Settings, Trash2 } from 'lucide-react';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export default function ConnectionSidebar({ 
  connections, 
  selectedConnection, 
  onSelectConnection 
}: {
  connections: any[];
  selectedConnection: string | null;
  onSelectConnection: (id: string) => void;
}) {
  const [showModal, setShowModal] = useState(false);
  const [newConnection, setNewConnection] = useState({
    name: '',
    type: 'postgresql' as const,
    host: '',
    port: '5432',
    database: '',
    username: '',
    password: '',
  });

  const handleCreateConnection = async () => {
    await prisma.connection.create({
      data: {
        ...newConnection,
        userId: 'current-user-id', // 从 session 获取
      },
    });
    setShowModal(false);
    // 刷新连接列表
  };

  return (
    <div className="w-64 bg-white border-r border-gray-200 flex flex-col">
      {/* 头部 */}
      <div className="p-4 border-b border-gray-200 flex justify-between items-center">
        <h2 className="font-semibold text-gray-800">Connections</h2>
        <button 
          onClick={() => setShowModal(true)}
          className="p-2 hover:bg-gray-100 rounded-lg"
        >
          <Plus className="w-5 h-5 text-blue-500" />
        </button>
      </div>

      {/* 连接列表 */}
      <div className="flex-1 overflow-y-auto p-2">
        {connections.map(conn => (
          <button
            key={conn.id}
            onClick={() => onSelectConnection(conn.id)}
            className={`w-full text-left px-3 py-2 rounded-lg flex items-center gap-3 mb-1 transition-colors ${
              selectedConnection === conn.id 
                ? 'bg-blue-50 text-blue-700' 
                : 'hover:bg-gray-50 text-gray-700'
            }`}
          >
            <Database className="w-4 h-4" />
            <span className="truncate">{conn.name}</span>
          </button>
        ))}
      </div>

      {/* 模态框 */}
      {showModal && (
        <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
          <div className="bg-white rounded-lg p-6 w-full max-w-md mx-4">
            <h3 className="text-lg font-semibold mb-4">Create Connection</h3>
            <div className="space-y-3">
              <input
                type="text"
                placeholder="Connection Name"
                value={newConnection.name}
                onChange={(e) => setNewConnection({...newConnection, name: e.target.value})}
                className="w-full px-3 py-2 border border-gray-300 rounded-lg"
              />
              {/* 添加其他字段... */}
            </div>
            <div className="flex gap-3 mt-6">
              <button
                onClick={() => setShowModal(false)}
                className="flex-1 px-4 py-2 border border-gray-300 rounded-lg"
              >
                Cancel
              </button>
              <button
                onClick={handleCreateConnection}
                className="flex-1 px-4 py-2 bg-blue-500 text-white rounded-lg"
              >
                Create
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}
```

## 运行项目

```bash
pnpm dev
```

## 功能扩展路线图

1. **基础功能**
    - [x] 用户认证
    - [x] 数据库连接管理
    - [x] 自然语言转 SQL
    - [x] SQL 执行和结果展示

2. **进阶功能**
    - [ ] 会话历史管理
    - [ ] 多数据库支持优化
    - [ ] SQL 语法高亮
    - [ ] 查询结果可视化

3. **高级功能**
    - [ ] 团队协作
    - [ ] 查询性能分析
    - [ ] 数据导出
    - [ ] 自定义模型支持（如 Ollama）

## 关键注意事项

1. **安全性**：数据库密码需要加密存储，不要明文保存
2. **错误处理**：完善的错误处理和用户友好的提示
3. **性能优化**：长连接管理、查询超时控制
4. **用户体验**：加载状态、动画效果、响应式设计

这是一个完整的从零构建指南。你可以根据实际需求逐步实现各个模块，先完成核心功能，再逐步添加进阶功能。










toolName: view_files

status: success


filePath: /Users/fangyaxi/00work/GithubProjects/fanxing/sqlchat-main/README.md

## SQL Chat 项目总结（ATS 法）

### **A - Aim（目的）**

**核心目标**：打造一款基于聊天界面的 SQL 客户端，让用户通过自然语言与数据库交互，实现查询、修改、增删等操作。

**背景意义**：进入「开发者工具 2.0 时代」，利用 AI 能力重构传统工具，以更直观的聊天界面替代繁琐的 UI 控件操作。

---

### **T - Task（任务/功能）**

| 功能模块 | 核心任务 | 技术支撑 |
|---------|---------|---------|
| **自然语言转 SQL** | 将用户自然语言提问转换为可执行 SQL | OpenAI API |
| **多数据库支持** | 兼容 MySQL、PostgreSQL、MSSQL、TiDB、OceanBase | 多数据库驱动 |
| **连接管理** | 创建、配置、管理数据库连接 | Prisma ORM |
| **会话系统** | 保存聊天历史，支持多会话切换 | 数据库存储 |
| **用户认证** | 支持账号系统、配额管理、支付功能 | NextAuth |
| **部署灵活性** | 支持单用户无数据库模式 / 多用户有数据库模式 | 环境变量配置 |

---

### **S - Solution（解决方案）**

#### **技术架构**
```
┌─────────────────────────────────────────────────────────────┐
│                      前端层 (Next.js)                       │
│  - 聊天界面组件          - 连接管理侧边栏          - 会话列表   │
├─────────────────────────────────────────────────────────────┤
│                      API 层 (Next.js API)                   │
│  - /api/chat          - /api/connection          - /api/auth │
├─────────────────────────────────────────────────────────────┤
│                      服务层                                  │
│  - OpenAI 服务        - 数据库连接服务        - 认证服务     │
├─────────────────────────────────────────────────────────────┤
│                      数据层                                  │
│  - PostgreSQL (应用数据)        - 用户数据库 (业务数据)        │
└─────────────────────────────────────────────────────────────┘
```

#### **关键技术栈**

| 分类 | 技术 | 版本 | 作用 |
|------|------|------|------|
| 框架 | Next.js | 13.x | SSR/SSG 支持，API 路由 |
| 数据库 | PostgreSQL | - | 存储应用数据 |
| ORM | Prisma | 4.x | 数据库操作 |
| 认证 | NextAuth | 4.x | 用户认证与会话管理 |
| AI | OpenAI API | - | 自然语言转 SQL |
| 样式 | Tailwind CSS | 3.x | 快速构建 UI |

#### **核心工作流程**
1. **用户提问** → 前端发送请求到 `/api/chat`
2. **获取 Schema** → 服务端查询目标数据库结构
3. **生成 SQL** → 调用 OpenAI API 将自然语言转为 SQL
4. **执行查询** → 连接目标数据库执行 SQL
5. **返回结果** → 将执行结果返回给前端展示

#### **部署方案**

| 模式 | 适用场景 | 配置文件 |
|------|---------|---------|
| **无数据库模式** | 个人使用 | `.env.nodb` |
| **有数据库模式** | 多用户服务 | `.env.usedb` |
| **Docker** | 快速部署 | `docker run` |

---

### **总结**

SQL Chat 是一款面向「开发者工具 2.0 时代」的 AI 驱动型数据库客户端，通过自然语言交互降低数据库操作门槛，支持多数据库类型，提供灵活的部署方案（单用户/多用户/Docker），核心价值在于将复杂的 SQL 操作转化为直观的聊天体验。
        