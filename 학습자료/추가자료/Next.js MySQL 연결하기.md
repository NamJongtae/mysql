### 0. 테이블 생성 및 데이터 삽입

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    nickname VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (name, nickname, email) 
VALUES 
('John Doe', 'johnd', 'john@example.com'),
('Jane Smith', 'janes', 'jane@example.com'),
('Alice Brown', 'aliceb', 'alice@example.com'),
('Bob Johnson', 'bobj', 'bob@example.com');
```

<br/>

### 1. 라이브러리 설치하기

```bash
npm install mysql2
npm install --save-dev @types/mysql
```

<br/>

### 2. MySQL 연결 및 쿼리 실행 함수

- MySQL과의 연결을 위한 풀(pool)을 생성하는 코드입니다.
- `registerService` **함수**: Next.js가 개발 모드에서 API 경로를 재구축할 때, 데이터베이스 연결 풀을 글로벌 캐시에 저장하여 중복 생성되지 않도록 관리합니다. 이로 인해 개발 환경에서 재연결을 방지할 수 있습니다.
- `executeQuery` **함수**: SQL 쿼리를 실행하고, 결과를 반환하는 함수입니다. 쿼리 실행 중 에러가 발생하면 콘솔에 출력하고, 에러 처리 후 Promise를 반환합니다.

```tsx
import { Pool, createPool } from 'mysql2';
/**
 * check, globalObject, registerService
 * Next.js는 개발 모드에서 API 경로를 지속적으로 재구축하는데, initFn()의 경로를 전역으로 지정하여 변경되지 않도록 합니다.
 */
function check(it: false | (Window & typeof globalThis) | typeof globalThis) {
  return it && it.Math === Math && it;
}

const globalObject =
  check(typeof window === 'object' && window) ||
  check(typeof self === 'object' && self) ||
  check(typeof global === 'object' && global) ||
  (() => {
    return this;
  })() ||
  Function('return this')();

const registerService = (name: string, initFn: any) => {
  if (process.env.NODE_ENV === 'development') {
    if (!(name in globalObject)) {
      globalObject[name] = initFn();
    }
    return globalObject[name];
  }
  return initFn();
};

export let db: Pool;

try {
  // "db"라는 이름으로 등록된 풀을 글로벌 캐시 또는 새로 생성
  db = registerService("db", () => {
    const pool = createPool({
      host: process.env.DB_HOST,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_DATABASE,
      port: 3306,
      waitForConnections: true,
      connectionLimit: 10, // 최대 연결 수 제한
      queueLimit: 0 // 대기열 제한 없음
    });
    console.log("Database pool created");
    return pool;
  });
} catch (error) {
  console.error("Failed to create database connection pool:", error);
  throw new Error("Failed to initialize database connection");
}

/**
 * executeQuery는 쿼리를 실행하고 결과를 반환하는 함수입니다.
 * 쿼리 실행 중 에러가 발생하면 에러 메시지를 출력합니다.
 *
 * @param query - 실행할 SQL 쿼리 문자열
 * @param values - 쿼리에 삽입할 값 배열
 * @returns Promise<any> - 쿼리 결과를 반환하는 Promise
 */
async function executeQuery(query: string, values: any[] = []) {
  if (!db) {
    throw new Error("DB has not been initialized");
  }

  // 쿼리 실행 및 에러 처리
  return new Promise((resolve, reject) => {
    db.query(query, values, (error, results) => {
      if (error) {
        console.error("Database query error:", error);
        return reject(error); // 에러 발생 시 Promise reject
      }
      resolve(results); // 성공 시 Promise resolve
    });
  });
}

export default executeQuery;

```

<br/>

### 3. 데이터 가져오기

이제 `Next.js` 환경에서 MySQL과 연동하여 사용자 데이터를 가져오는 예시 컴포넌트를 만들겠습니다. 이 컴포넌트는 서버 측에서 데이터를 가져오며, 간단한 스타일을 추가해 테이블을 렌더링합니다.

```tsx
import executeQuery from "@/lib/db"; // 위에서 만든 executeQuery 함수

// 서버 구성 요소 - 사용자 데이터를 DB에서 가져오는 함수
export default async function UsersPage() {
  let users = [] as {
    id: number;
    name: string;
    nickname: string;
    email: string;
    createdAt: string;
  }[];

  try {
    // MySQL에서 사용자 데이터를 가져오는 쿼리 실행
    users = (await executeQuery("SELECT * FROM users", [])) as {
      id: number;
      name: string;
      nickname: string;
      email: string;
      createdAt: string;
    }[];
  } catch (error) {
    console.error("Failed to load users:", error);
  }

  return (
    <div>
      <div style={styles.wrapper}>
        <h2 style={styles.heading}>Users table</h2>
        <table style={styles.table}>
          <thead>
            <tr style={styles.headerRow}>
              <th style={styles.headerCell}>ID</th>
              <th style={styles.headerCell}>Name</th>
              <th style={styles.headerCell}>Nickname</th>
              <th style={styles.headerCell}>Email</th>
              <th style={styles.headerCell}>Created At</th>
            </tr>
          </thead>
          <tbody>
            {users.map((user) => (
              <tr key={user.id} style={styles.row}>
                <td style={styles.cell}>{user.id}</td>
                <td style={styles.cell}>{user.name}</td>
                <td style={styles.cell}>{user.nickname}</td>
                <td style={styles.cell}>{user.email}</td>
                <td style={styles.cell}>
                  {new Date(user.createdAt).toLocaleDateString()}
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}

const styles = {
  wrapper: {
    width: "100%",
    maxWidth: "1080px",
    margin: "0 auto",
    padding: "0 20px",
  },
  heading: {
    textAlign: "center" as "center",
    margin: "50px 0 20px 0",
    fontSize: "24px",
    fontWeight: "bold",
  },
  table: {
    width: "100%",
    borderCollapse: "collapse" as "collapse",
    boxShadow: "0 2px 10px rgba(0, 0, 0, 0.1)",
    borderRadius: "10px",
    overflow: "hidden",
  },
  headerRow: {
    backgroundColor: "#0070f3",
    color: "white",
  },
  headerCell: {
    padding: "12px",
    textAlign: "left" as "left",
    fontSize: "14px",
    fontWeight: "bold" as "bold",
    borderBottom: "2px solid #ddd",
  },
  row: {
    backgroundColor: "#f9f9f9",
    transition: "background-color 0.3s",
  },
  cell: {
    padding: "10px",
    textAlign: "left" as "left",
    borderBottom: "1px solid #ddd",
  },
  rowHover: {
    backgroundColor: "#f1f1f1",
  },
};

```

<br/>

### 4. Server Action 사용하기

```tsx
// app/server-actions/actions.ts

'use server';

import executeQuery from '@/lib/db';

/**
 * Server Action 함수로 MySQL 데이터베이스에서 사용자를 추가하는 함수
 * @param name - 사용자 이름
 * @param nickname - 사용자 닉네임
 * @param email - 사용자 이메일
 */
export async function addUser(name: string, nickname: string, email: string) {
  try {
    // INSERT 쿼리를 실행해 사용자 추가
    await executeQuery(
      'INSERT INTO users (name, nickname, email) VALUES (?, ?, ?)',
      [name, nickname, email]
    );
    console.log("User added successfully");
  } catch (error) {
    console.error('Error adding user:', error);
    throw new Error('Failed to add user');
  }
}

/**
 * Server Action 함수로 MySQL 데이터베이스에서 모든 사용자 데이터를 가져오는 함수
 */
export async function fetchUsers() {
  try {
    // SELECT 쿼리를 실행해 모든 사용자 데이터를 가져옴
    const users = await executeQuery(
      'SELECT id, name, nickname, email, createdAt FROM users',
      []
    );
    return users;
  } catch (error) {
    console.error('Error fetching users:', error);
    throw new Error('Failed to fetch users');
  }
}

```

```tsx
// app/src/component/add-user-form.tsx

"use client";

import { useState } from "react";
import { addUser } from "@/server-actions/actions";

export default function AddUserForm() {
  const [name, setName] = useState("");
  const [nickname, setNickname] = useState("");
  const [email, setEmail] = useState("");

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();

    // 서버 액션 호출하여 새로운 사용자 추가
    await addUser(name, nickname, email);

    // 입력 필드 초기화
    setName("");
    setNickname("");
    setEmail("");

    // 페이지를 새로고침하여 최신 사용자 목록 표시
    window.location.reload();
  };

  return (
    <>
      <h2 style={styles.heading}>User Form</h2>
      <form onSubmit={handleSubmit} style={styles.form}>
        <div style={styles.formGroup}>
          <label htmlFor="name" style={styles.label}>
            Name:
          </label>
          <input
            type="text"
            id="name"
            value={name}
            onChange={(e) => setName(e.target.value)}
            style={styles.input}
            required
          />
        </div>
        <div style={styles.formGroup}>
          <label htmlFor="nickname" style={styles.label}>
            Nickname:
          </label>
          <input
            type="text"
            id="nickname"
            value={nickname}
            onChange={(e) => setNickname(e.target.value)}
            style={styles.input}
            required
          />
        </div>
        <div style={styles.formGroup}>
          <label htmlFor="email" style={styles.label}>
            Email:
          </label>
          <input
            type="email"
            id="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            style={styles.input}
            required
          />
        </div>
        <button type="submit" style={styles.button}>
          Add User
        </button>
      </form>
    </>
  );
}

const styles = {
  heading: {
    textAlign: "center" as "center",
    margin: "20px 0",
    fontSize: "24px",
    fontWeight: "bold",
  },
  form: {
    display: "flex",
    flexDirection: "column" as "column",
    width: "100%",
    maxWidth: "400px",
    margin: "0 auto",
    padding: "20px",
    backgroundColor: "#f9f9f9",
    borderRadius: "8px",
    boxShadow: "0 2px 10px rgba(0, 0, 0, 0.1)",
  },
  formGroup: {
    marginBottom: "15px",
  },
  label: {
    fontSize: "14px",
    fontWeight: "bold" as "bold",
    marginBottom: "5px",
    color: "#333",
  },
  input: {
    width: "100%",
    padding: "10px",
    fontSize: "16px",
    border: "1px solid #ddd",
    borderRadius: "4px",
    transition: "border-color 0.3s",
  },
  inputFocus: {
    borderColor: "#0070f3",
    outline: "none",
  },
  button: {
    width: "100%",
    padding: "10px 15px",
    fontSize: "16px",
    backgroundColor: "#0070f3",
    color: "white",
    border: "none",
    borderRadius: "4px",
    cursor: "pointer",
    transition: "background-color 0.3s",
  },
  buttonHover: {
    backgroundColor: "#005bb5",
  },
};

```

```tsx
// app/src/component/users-table.tsx

interface IProps {
  users: {
    id: number;
    name: string;
    nickname: string;
    email: string;
    createdAt: string;
  }[];
}

export default function UsersTable({ users }: IProps) {
  return (
    <div style={styles.wrapper}>
      <h2 style={styles.heading}>Users table</h2>
      <table style={styles.table}>
        <thead>
          <tr style={styles.headerRow}>
            <th style={styles.headerCell}>ID</th>
            <th style={styles.headerCell}>Name</th>
            <th style={styles.headerCell}>Nickname</th>
            <th style={styles.headerCell}>Email</th>
            <th style={styles.headerCell}>Created At</th>
          </tr>
        </thead>
        <tbody>
          {users.map((user) => (
            <tr key={user.id} style={styles.row}>
              <td style={styles.cell}>{user.id}</td>
              <td style={styles.cell}>{user.name}</td>
              <td style={styles.cell}>{user.nickname}</td>
              <td style={styles.cell}>{user.email}</td>
              <td style={styles.cell}>
                {new Date(user.createdAt).toLocaleDateString()}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

const styles = {
  wrapper: {
    width:"100%",
    maxWidth:"1080px",
    margin: "0 auto",
    padding:"0 20px"
  },
  heading: {
    textAlign: "center" as "center",
    margin: "50px 0 20px 0",
    fontSize: "24px",
    fontWeight: "bold",
  },
  table: {
    width: "100%",
    borderCollapse: "collapse" as "collapse",
    boxShadow: "0 2px 10px rgba(0, 0, 0, 0.1)",
    borderRadius:"10px",
    overflow:"hidden"
  },
  headerRow: {
    backgroundColor: "#0070f3",
    color: "white",
  },
  headerCell: {
    padding: "12px",
    textAlign: "left" as "left",
    fontSize: "14px",
    fontWeight: "bold" as "bold",
    borderBottom: "2px solid #ddd",
  },
  row: {
    backgroundColor: "#f9f9f9",
    transition: "background-color 0.3s",
  },
  cell: {
    padding: "10px",
    textAlign: "left" as "left",
    borderBottom: "1px solid #ddd",
  },
  rowHover: {
    backgroundColor: "#f1f1f1",
  },
};

```

```tsx
// app/users/page.tsx

import { useState } from 'react';
import { addUser, fetchUsers } from '@/server-actions/actions';

export default async function UsersPage() {
  // 서버 액션을 사용해 사용자 데이터를 가져옴
  const users = await fetchUsers();

  return (
    <div>
      <AddUserForm />
      <UsersTable users={users} />
    </div>
  );
}
```