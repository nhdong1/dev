# Sequelize — Models, Associations và Hooks

> Sequelize là ORM (Object-Relational Mapping — Ánh Xạ Đối Tượng–Quan Hệ) lâu đời nhất trong Node.js ecosystem. Vẫn gặp trong legacy projects và một số enterprise codebases.

## Mục Lục

1. [Sequelize Là Gì](#sequelize-là-gì)
2. [Khởi Tạo và Model Definition](#khởi-tạo-và-model-definition)
3. [CRUD Operations](#crud-operations)
4. [Associations (Quan Hệ)](#associations-quan-hệ)
5. [Hooks (Lifecycle Callbacks)](#hooks-lifecycle-callbacks)
6. [Migrations với sequelize-cli](#migrations-với-sequelize-cli)
7. [Scopes và Validations](#scopes-và-validations)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Sequelize Là Gì

| Tính Năng | Mô Tả |
| --------- | ----- |
| **Model-based** | Định nghĩa models với `define()` hoặc class extends `Model` |
| **Associations** | `hasMany`, `belongsTo`, `belongsToMany` |
| **Hooks** | `beforeCreate`, `afterUpdate` — lifecycle callbacks |
| **Migrations** | `sequelize-cli` quản lý schema versions |
| **Transactions** | Built-in transaction support |

```bash
npm install sequelize pg pg-hstore
npm install --save-dev sequelize-cli
npx sequelize-cli init
```

---

## Khởi Tạo và Model Definition

```javascript
const { Sequelize, DataTypes, Model } = require('sequelize');

const sequelize = new Sequelize(process.env.DATABASE_URL, {
  dialect: 'postgres',
  logging: process.env.NODE_ENV === 'development' ? console.log : false,
  pool: {
    max: 20,
    min: 0,
    acquire: 30000,
    idle: 10000,
  },
});

// Class-based model (Sequelize v6+)
class User extends Model {}

User.init(
  {
    id: {
      type: DataTypes.UUID,
      defaultValue: DataTypes.UUIDV4,
      primaryKey: true,
    },
    email: {
      type: DataTypes.STRING,
      allowNull: false,
      unique: true,
      validate: { isEmail: true },
    },
    name: DataTypes.STRING,
    role: {
      type: DataTypes.ENUM('USER', 'ADMIN'),
      defaultValue: 'USER',
    },
  },
  {
    sequelize,
    modelName: 'User',
    tableName: 'users',
    underscored: true, // created_at thay vì createdAt
    timestamps: true,
  }
);

class Post extends Model {}

Post.init(
  {
    id: {
      type: DataTypes.UUID,
      defaultValue: DataTypes.UUIDV4,
      primaryKey: true,
    },
    title: { type: DataTypes.STRING, allowNull: false },
    content: DataTypes.TEXT,
    published: { type: DataTypes.BOOLEAN, defaultValue: false },
  },
  { sequelize, modelName: 'Post', tableName: 'posts', underscored: true }
);
```

---

## CRUD Operations

```javascript
// CREATE
const user = await User.create({ email: 'alice@example.com', name: 'Alice' });

// READ
const users = await User.findAll({
  where: { role: 'USER' },
  order: [['created_at', 'DESC']],
  limit: 20,
  offset: 0,
  attributes: ['id', 'email', 'name'], // exclude password
});

const user = await User.findByPk(userId);
const user = await User.findOne({ where: { email: 'alice@example.com' } });

// UPDATE
await user.update({ name: 'Alice Updated' });
// hoặc
await User.update({ role: 'ADMIN' }, { where: { id: userId } });

// DELETE
await user.destroy();
await User.destroy({ where: { role: 'USER' } });

// UPSERT
const [user, created] = await User.findOrCreate({
  where: { email: 'bob@example.com' },
  defaults: { name: 'Bob' },
});
```

### Query Operators

```javascript
const { Op } = require('sequelize');

await User.findAll({
  where: {
    [Op.and]: [
      { email: { [Op.like]: '%@gmail.com' } },
      { created_at: { [Op.gte]: new Date('2024-01-01') } },
    ],
    [Op.or]: [
      { role: 'ADMIN' },
      { name: { [Op.ne]: null } },
    ],
  },
});
```

---

## Associations (Quan Hệ)

```javascript
// One-to-Many: User hasMany Posts
User.hasMany(Post, { foreignKey: 'author_id', as: 'posts' });
Post.belongsTo(User, { foreignKey: 'author_id', as: 'author' });

// Many-to-Many: Post belongsToMany Tags
Post.belongsToMany(Tag, { through: 'post_tags', foreignKey: 'post_id' });
Tag.belongsToMany(Post, { through: 'post_tags', foreignKey: 'tag_id' });

// Eager loading
const users = await User.findAll({
  include: [
    {
      model: Post,
      as: 'posts',
      where: { published: true },
      required: false, // LEFT JOIN (false) vs INNER JOIN (true)
    },
  ],
});

// Nested create
await User.create(
  {
    email: 'charlie@example.com',
    posts: [{ title: 'First Post' }, { title: 'Second Post' }],
  },
  { include: [Post] }
);
```

### Association Types

| Method | Quan Hệ | FK Location |
| ------ | ------- | ----------- |
| `hasMany` / `belongsTo` | 1:N | `belongsTo` side |
| `hasOne` / `belongsTo` | 1:1 | `belongsTo` side |
| `belongsToMany` | N:N | Join table |

---

## Hooks (Lifecycle Callbacks)

```javascript
User.addHook('beforeCreate', async (user) => {
  if (user.password) {
  const bcrypt = require('bcrypt');
    user.password = await bcrypt.hash(user.password, 12);
  }
});

User.addHook('afterUpdate', async (user, options) => {
  console.log(`User ${user.id} updated`);
});

// Hoặc trong model definition
User.init({ /* ... */ }, {
  hooks: {
    beforeValidate: (user) => {
      if (user.email) user.email = user.email.toLowerCase();
    },
    afterDestroy: async (user) => {
      await AuditLog.create({ action: 'DELETE', entityId: user.id });
    },
  },
});
```

### Hook Order

```
beforeValidate → validate → beforeCreate → create → afterCreate
beforeUpdate → update → afterUpdate
beforeDestroy → destroy → afterDestroy
```

---

## Migrations với sequelize-cli

```bash
npx sequelize-cli migration:generate --name create-users-table
npx sequelize-cli db:migrate
npx sequelize-cli db:migrate:undo
```

```javascript
// migrations/20240101000000-create-users-table.js
'use strict';

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable('users', {
      id: {
        type: Sequelize.UUID,
        defaultValue: Sequelize.UUIDV4,
        primaryKey: true,
      },
      email: {
        type: Sequelize.STRING,
        allowNull: false,
        unique: true,
      },
      name: Sequelize.STRING,
      created_at: { type: Sequelize.DATE, allowNull: false },
      updated_at: { type: Sequelize.DATE, allowNull: false },
    });
    await queryInterface.addIndex('users', ['email']);
  },

  async down(queryInterface) {
    await queryInterface.dropTable('users');
  },
};
```

> **Không dùng `sequelize.sync()` trong production** — dùng migrations.

---

## Scopes và Validations

### Default Scopes

```javascript
User.addScope('active', {
  where: { deleted_at: null },
});

User.addScope('withPosts', {
  include: [{ model: Post, as: 'posts' }],
});

// Sử dụng
const activeUsers = await User.scope('active').findAll();
const usersWithPosts = await User.scope(['active', 'withPosts']).findAll();
```

### Model Validations

```javascript
email: {
  type: DataTypes.STRING,
  validate: {
    isEmail: { msg: 'Invalid email format' },
    notEmpty: { msg: 'Email is required' },
    async isUnique(value) {
      const existing = await User.findOne({ where: { email: value } });
      if (existing) throw new Error('Email already exists');
    },
  },
},
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Sequelize vs Prisma — khi nào migrate?

**Trả lời:** Migrate khi greenfield project hoặc team muốn type safety tốt hơn. Giữ Sequelize khi codebase lớn, migrations phức tạp, hoặc team đã thành thạo. Migration cost cao — không migrate chỉ vì "trend".

### Câu 2: `include` với `required: false` vs `true`?

**Trả lời:** `required: false` → LEFT OUTER JOIN — trả parent kể cả không có children. `required: true` → INNER JOIN — chỉ parent có matching children.

### Câu 3: Hooks vs Service layer logic?

**Trả lời:** Hooks cho cross-cutting concerns (hash password, audit log). Business logic nên ở service layer — hooks khó test và debug, ẩn side effects.

---

**Xem tiếp:** [5-mongodb-mongoose.md](./5-mongodb-mongoose.md) — ODM cho MongoDB.
