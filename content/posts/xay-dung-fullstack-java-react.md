---
title: "Xây dựng ứng dụng Fullstack với Java Spring Boot và ReactJS – Hướng dẫn chi tiết từng bước "
date: 2025-10-22
draft: false
description: "Hướng dẫn từng bước xây dựng ứng dụng quản lý người dùng bằng Java (Spring Boot) cho backend và ReactJS cho frontend — kèm mã mẫu sẵn để copy/run."
tags: ["Java", "Spring Boot", "React", "Fullstack", "API", "Web Development", "MySQL"]
categories: ["Lập trình", "Hướng dẫn", "Fullstack"]
---

# 🧭 Giới thiệu

Bài hướng dẫn này đưa bạn qua các bước thực tế để tạo một ứng dụng quản lý người dùng (User Management) gồm:
- Backend: Spring Boot (REST API, JPA, MySQL)
- Frontend: React (Vite hoặc Create React App)
- (Tùy chọn) Docker Compose để chạy MySQL + backend

Mục tiêu: mã mẫu dễ copy/run, giải thích ngắn gọn để bạn mở rộng.

---

## 🧱 1. Cấu trúc dự án (gợi ý)

project/
```
backend/   # Spring Boot (Maven)
frontend/  # React (Vite hoặc CRA)
docker-compose.yml (tùy chọn)
```

---

## ⚙️ 2. Backend — Spring Boot (các file chính và lưu ý)

Dependencies cốt lõi (pom.xml): spring-boot-starter-web, spring-boot-starter-data-jpa, mysql-connector-java, spring-boot-starter-validation, lombok (tuỳ chọn), spring-boot-devtools (dev).

### application.properties (khuyến nghị dùng env vars)
```properties
spring.datasource.url=${SPRING_DATASOURCE_URL:jdbc:mysql://localhost:3306/userdb?useSSL=false&serverTimezone=UTC}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME:root}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:password}
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
server.port=8080
```

Gợi ý: không để mật khẩu cứng trong repo; dùng biến môi trường khi deploy.

### Entity: User.java
```java
package com.example.backend.model;

import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import lombok.*;

@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    @Id
    @Email
    private String email;

    @NotBlank
    private String name;
}
```

### Repository: UserRepository.java
```java
package com.example.backend.repository;

import com.example.backend.model.User;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<User, String> { }
```

### Service: UserService.java
```java
package com.example.backend.service;

import com.example.backend.model.User;
import com.example.backend.repository.UserRepository;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class UserService {
    private final UserRepository repo;

    public UserService(UserRepository repo) {
        this.repo = repo;
    }

    public List<User> findAll() {
        return repo.findAll();
    }

    public User save(User user) {
        return repo.save(user);
    }
}
```

### Controller: UserController.java (REST API)
```java
package com.example.backend.controller;

import com.example.backend.model.User;
import com.example.backend.service.UserService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import jakarta.validation.Valid;
import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/api/users")
@CrossOrigin(origins = "http://localhost:3000") // thay origin khi deploy
public class UserController {
    private final UserService service;

    public UserController(UserService service) {
        this.service = service;
    }

    @GetMapping
    public List<User> getAll() {
        return service.findAll();
    }

    @PostMapping
    public ResponseEntity<User> create(@Valid @RequestBody User user) {
        User saved = service.save(user);
        return ResponseEntity.created(URI.create("/api/users/" + saved.getEmail())).body(saved);
    }
}
```

(Tùy chọn) Thêm ControllerAdvice để trả lỗi validate thân thiện.

### Chạy backend (Windows)
- Dùng Maven wrapper (nếu có): .\mvnw.cmd spring-boot:run  
- Hoặc: mvn spring-boot:run  
API mặc định: http://localhost:8080/api/users

---

## 🐳 3. Docker Compose (tùy chọn) — MySQL + backend

Tệp docker-compose.yml mẫu để dev nhanh:

```yaml
version: "3.8"
services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: userdb
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/userdb?useSSL=false&serverTimezone=UTC
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: password
    depends_on:
      db:
        condition: service_healthy
```

Chạy: docker-compose up --build

---

## 🌐 4. Frontend — React (Vite/Cra) — mã mẫu

Tạo nhanh:
- Vite: npm create vite@latest frontend -- --template react  
- CRA: npx create-react-app frontend

cd frontend && npm install && npm run dev (Vite) / npm start (CRA)

### src/components/UserList.jsx
```javascript
import React, { useEffect, useState } from "react";

export default function UserList({ refreshSignal }) {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const fetchUsers = () => {
    setLoading(true);
    fetch("http://localhost:8080/api/users")
      .then((res) => {
        if (!res.ok) throw new Error("Network response was not ok");
        return res.json();
      })
      .then((data) => setUsers(data))
      .catch((err) => setError(err.message))
      .finally(() => setLoading(false));
  };

  useEffect(() => {
    fetchUsers();
  }, [refreshSignal]);

  if (loading) return <div>Đang tải...</div>;
  if (error) return <div style={{ color: "red" }}>Lỗi: {error}</div>;

  return (
    <ul>
      {users.map((u) => (
        <li key={u.email}>
          <strong>{u.name}</strong> — {u.email}
        </li>
      ))}
    </ul>
  );
}
```

### src/components/AddUser.jsx
```javascript
import React, { useState } from "react";

export default function AddUser({ onAdded }) {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState(null);

  const submit = async (e) => {
    e.preventDefault();
    setSaving(true);
    setError(null);
    try {
      const res = await fetch("http://localhost:8080/api/users", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ name, email }),
      });
      if (!res.ok) {
        const text = await res.text();
        throw new Error(text || "Lỗi khi lưu user");
      }
      setName("");
      setEmail("");
      onAdded(); // báo parent refresh
    } catch (err) {
      setError(err.message);
    } finally {
      setSaving(false);
    }
  };

  return (
    <form onSubmit={submit} style={{ marginBottom: 16 }}>
      <div>
        <input placeholder="Name" value={name} onChange={(e) => setName(e.target.value)} required />
      </div>
      <div>
        <input placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} type="email" required />
      </div>
      <button type="submit" disabled={saving}>{saving ? "Đang lưu..." : "Thêm người dùng"}</button>
      {error && <div style={{ color: "red" }}>{error}</div>}
    </form>
  );
}
```

### src/App.jsx
```javascript
import React, { useState } from "react";
import UserList from "./components/UserList";
import AddUser from "./components/AddUser";

export default function App() {
  const [signal, setSignal] = useState(0);
  const onAdded = () => setSignal((s) => s + 1);

  return (
    <div style={{ padding: 20 }}>
      <h1>User Management</h1>
      <AddUser onAdded={onAdded} />
      <UserList refreshSignal={signal} />
    </div>
  );
}
```

Gợi ý: sử dụng React Query / SWR để quản lý cache và refetch thay vì signal.

---

## 🔗 5. CORS & kết nối

- Nếu frontend chạy port 3000 và backend 8080, bật CORS (ví dụ @CrossOrigin hoặc cấu hình WebMvcConfigurer).  
- Trong production, chỉ cho phép origin đáng tin cậy.

---

## ✅ 6. Hướng dẫn chạy (Windows)

1. Backend:
   - Tạo project Spring Boot (start.spring.io) hoặc clone template.
   - Thêm code ở trên vào package tương ứng.
   - Chạy: .\mvnw.cmd spring-boot:run  (hoặc mvn spring-boot:run)
   - Kiểm tra: curl http://localhost:8080/api/users

2. Frontend:
   - cd frontend
   - npm install
   - npm run dev (Vite) hoặc npm start (CRA)
   - Mở http://localhost:3000

3. Docker (tùy chọn):
   - docker-compose up --build

---

## ⚡ Nâng cấp đề xuất (các bước tiếp theo)
- Thêm validation/ControllerAdvice cho lỗi rõ ràng.  
- Spring Security + JWT cho authentication.  
- Pagination/sorting ở API; DTO mapping (MapStruct).  
- Tests: JUnit + MockMvc (backend), Jest + React Testing Library (frontend).  
- Dockerize từng service, thêm CI/CD.

---

## 🔍 Tóm tắt

Tập trung:
- Backend: API đơn giản, JPA, validation, CORS.
- Frontend: component tách biệt, xử lý loading/error, refresh sau POST.
- Chạy local: đảm bảo backend chạy trước, bật CORS, rồi frontend.

