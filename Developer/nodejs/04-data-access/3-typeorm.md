# TypeORM — Entity, Repository và Query Builder

> TypeORM là ORM (Object-Relational Mapping — Ánh Xạ Đối Tượng–Quan Hệ) dựa trên decorators, tích hợp sâu với NestJS. Hỗ trợ Active Record và Data Mapper patterns.

## Mục Lục

1. [TypeORM Là Gì](#typeorm-là-gì)
2. [Entity Definition](#entity-definition)
3. [DataSource và Connection](#datasource-và-connection)
4. [Repository Pattern](#repository-pattern)
5. [Query Builder](#query-builder)
6. [Relations và Eager Loading](#relations-và-eager-loading)
7. [Migrations](#migrations)
8. [NestJS Integration](#nestjs-integration)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## TypeORM Là Gì

| Tính Năng | Mô Tả |
| --------- | ----- |
| **Decorators** | `@Entity`, `@Column`, `@ManyToOne` — quen thuộc với Java/Spring devs |
| **Active Record** | Entity extends `BaseEntity`, gọi `user.save()` trực tiếp |
| **Data Mapper** | Repository pattern — tách persistence khỏi domain |
| **Query Builder** | Fluent SQL builder cho complex queries |
| **Multi-DB** | PostgreSQL, MySQL, SQLite, MongoDB |

```bash
npm install typeorm reflect-metadata pg
# TypeScript decorators cần trong tsconfig.json:
# "experimentalDecorators": true, "emitDecoratorMetadata": true
```

---

## Entity Definition

```typescript
import {
  Entity, PrimaryGeneratedColumn, Column, CreateDateColumn,
  UpdateDateColumn, OneToMany, ManyToOne, JoinColumn, Index,
} from 'typeorm';

@Entity('users')
@Index(['email'])
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  email: string;

  @Column({ nullable: true })
  name: string;

  @Column({ type: 'enum', enum: ['USER', 'ADMIN'], default: 'USER' })
  role: string;

  @OneToMany(() => Post, (post) => post.author)
  posts: Post[];

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}

@Entity('posts')
export class Post {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  title: string;

  @Column({ type: 'text', nullable: true })
  content: string;

  @Column({ default: false })
  published: boolean;

  @Column({ name: 'author_id' })
  authorId: string;

  @ManyToOne(() => User, (user) => user.posts, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'author_id' })
  author: User;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

---

## DataSource và Connection

```typescript
import { DataSource } from 'typeorm';
import { User } from './entities/User';
import { Post } from './entities/Post';

export const AppDataSource = new DataSource({
  type: 'postgres',
  host: process.env.DB_HOST,
  port: 5432,
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  entities: [User, Post],
  migrations: ['src/migrations/*.ts'],
  synchronize: false, // ❌ KHÔNG bật trong production
  logging: process.env.NODE_ENV === 'development',
});

// Khởi tạo khi app start
AppDataSource.initialize()
  .then(() => console.log('DataSource initialized'))
  .catch((err) => console.error('DataSource error:', err));
```

> **`synchronize: true`** tự động sync schema — chỉ dùng prototype, có thể mất data trong production.

---

## Repository Pattern

```typescript
import { AppDataSource } from './data-source';
import { User } from './entities/User';

const userRepository = AppDataSource.getRepository(User);

// CREATE
const user = userRepository.create({ email: 'alice@example.com', name: 'Alice' });
await userRepository.save(user);

// READ
const users = await userRepository.find({
  where: { role: 'USER' },
  order: { createdAt: 'DESC' },
  take: 20,
  skip: 0,
});

const userById = await userRepository.findOne({
  where: { id: userId },
  relations: ['posts'],
});

// UPDATE
await userRepository.update(userId, { name: 'Alice Updated' });

// DELETE
await userRepository.delete(userId);

// COUNT
const count = await userRepository.count({ where: { role: 'ADMIN' } });
```

### Custom Repository

```typescript
@EntityRepository(User) // Legacy — dùng extend Repository trong TypeORM 0.3+
export class UserRepository extends Repository<User> {
  async findByEmailWithPosts(email: string): Promise<User | null> {
    return this.findOne({
      where: { email },
      relations: ['posts'],
    });
  }
}
```

---

## Query Builder

Cho queries phức tạp mà Repository API không đủ:

```typescript
const users = await userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post')
  .where('user.role = :role', { role: 'USER' })
  .andWhere('post.published = :published', { published: true })
  .orderBy('user.createdAt', 'DESC')
  .take(20)
  .getMany();

// Aggregation
const stats = await userRepository
  .createQueryBuilder('user')
  .leftJoin('user.posts', 'post')
  .select('user.id', 'id')
  .addSelect('COUNT(post.id)', 'postCount')
  .groupBy('user.id')
  .having('COUNT(post.id) > :min', { min: 5 })
  .getRawMany();

// Subquery
const subQuery = userRepository
  .createQueryBuilder('u')
  .select('u.id')
  .where('u.role = :role', { role: 'ADMIN' });

const posts = await postRepository
  .createQueryBuilder('post')
  .where(`post.author_id IN (${subQuery.getQuery()})`)
  .setParameters(subQuery.getParameters())
  .getMany();
```

---

## Relations và Eager Loading

### Relation Decorators

| Decorator | Quan Hệ | Ví Dụ |
| --------- | ------- | ----- |
| `@OneToOne` | 1:1 | User ↔ Profile |
| `@OneToMany` / `@ManyToOne` | 1:N | User → Posts |
| `@ManyToMany` | N:N | Post ↔ Tags |

### Eager vs Lazy Loading

```typescript
// Eager — luôn load relation (cẩn thận N+1 ngược: load tất cả khi không cần)
@ManyToOne(() => User, (user) => user.posts, { eager: true })
author: User;

// Explicit eager loading trong query
const users = await userRepository.find({
  relations: ['posts', 'posts.tags'],
});

// Query Builder
.leftJoinAndSelect('user.posts', 'post')  // JOIN + SELECT
.leftJoin('user.posts', 'post')           // JOIN only, không load vào entity
```

---

## Migrations

```bash
# Tạo migration từ entity changes
npx typeorm migration:generate -d src/data-source.ts src/migrations/AddUserRole

# Chạy migrations
npx typeorm migration:run -d src/data-source.ts

# Revert migration cuối
npx typeorm migration:revert -d src/data-source.ts
```

```typescript
// src/migrations/1234567890-AddUserRole.ts
import { MigrationInterface, QueryRunner } from 'typeorm';

export class AddUserRole1234567890 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      ALTER TABLE users ADD COLUMN role VARCHAR(10) DEFAULT 'USER'
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE users DROP COLUMN role`);
  }
}
```

---

## NestJS Integration

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './entities/user.entity';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST,
      entities: [User],
      synchronize: false,
    }),
    TypeOrmModule.forFeature([User]), // Register repository cho module
  ],
  controllers: [UsersController],
  providers: [UsersService],
})
export class AppModule {}

// users.service.ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>,
  ) {}

  findAll(): Promise<User[]> {
    return this.userRepository.find({ relations: ['posts'] });
  }
}
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Active Record vs Data Mapper trong TypeORM?

**Trả lời:** Active Record: entity có methods `save()`, `remove()` — đơn giản, phù hợp small apps. Data Mapper: dùng Repository tách persistence — phù hợp domain-driven design, testability cao hơn.

### Câu 2: TypeORM vs Prisma?

**Trả lời:** TypeORM: decorators, linh hoạt hơn, NestJS native. Prisma: schema DSL, type generation tốt hơn, migrations ổn định hơn, DX cao. Prisma cho greenfield TS; TypeORM cho NestJS enterprise hoặc Java-background teams.

### Câu 3: `leftJoin` vs `leftJoinAndSelect`?

**Trả lời:** `leftJoin` chỉ JOIN để filter/aggregate, không populate relation vào entity. `leftJoinAndSelect` JOIN và load data vào entity — dùng để eager load, tránh N+1.

---

**Xem tiếp:** [4-sequelize.md](./4-sequelize.md) — ORM legacy phổ biến trong Node.js codebase cũ.
