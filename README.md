# Sistema-de-Controle-de-Acesso
Sistema para Controle de Acesso de Usuários na Empresa
---

## Estrutura do projeto com arquivos de exemplo

Abaixo estão os arquivos principais já prontos para serem utilizados:

---

### `src/config/database.js`

```js
const { Sequelize } = require('sequelize');
const path = require('path');

const dbPath = process.env.DB_PATH || path.join(__dirname, '..', '..', 'database.sqlite');

const sequelize = new Sequelize({
  dialect: 'sqlite',
  storage: dbPath,
  logging: false,
});

module.exports = sequelize;
```

---

### `src/models/User.js`

```js
const { DataTypes } = require('sequelize');
const bcrypt = require('bcrypt');

module.exports = (sequelize) => {
  const User = sequelize.define('User', {
    id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
    name: { type: DataTypes.STRING, allowNull: false },
    email: { type: DataTypes.STRING, allowNull: false, unique: true, validate: { isEmail: true } },
    password: { type: DataTypes.STRING, allowNull: false },
  });

  User.beforeCreate(async (user) => {
    const rounds = parseInt(process.env.BCRYPT_SALT_ROUNDS) || 10;
    user.password = await bcrypt.hash(user.password, rounds);
  });

  User.prototype.validatePassword = function (password) {
    return bcrypt.compare(password, this.password);
  };

  return User;
};
```

---

### `src/models/index.js`

```js
const sequelize = require('../config/database');
const defineUser = require('./User');

const User = defineUser(sequelize);

module.exports = { sequelize, User };
```

---

### `src/app.js`

```js
require('dotenv').config();
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const morgan = require('morgan');

const { sequelize } = require('./models');
const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/users');

const app = express();

app.use(helmet());
app.use(express.json());
app.use(cors());
app.use(morgan('dev'));

app.use('/api/auth', authRoutes);
app.use('/api/users', userRoutes);

app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ message: 'Erro interno do servidor' });
});

const PORT = process.env.PORT || 3000;

sequelize.sync().then(() => {
  app.listen(PORT, () => console.log(`Servidor rodando na porta ${PORT}`));
});
```

---

### `src/routes/auth.js`

```js
const express = require('express');
const { body } = require('express-validator');
const { register, login } = require('../controllers/authController');

const router = express.Router();

router.post('/register', [
  body('name').notEmpty().withMessage('nome é obrigatório'),
  body('email').isEmail().withMessage('email inválido'),
  body('password').isLength({ min: 6 }).withMessage('senha muito curta'),
], register);

router.post('/login', [
  body('email').isEmail(),
  body('password').notEmpty(),
], login);

module.exports = router;
```

---

### `src/controllers/authController.js`

```js
const { validationResult } = require('express-validator');
const { User } = require('../models');
const jwt = require('jsonwebtoken');

exports.register = async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });

  const { name, email, password } = req.body;
  try {
    const existing = await User.findOne({ where: { email } });
    if (existing) return res.status(409).json({ message: 'Email já cadastrado' });

    const user = await User.create({ name, email, password });
    const { password: _, ...userData } = user.toJSON();
    res.status(201).json({ user: userData });
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Erro no servidor' });
  }
};

exports.login = async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });

  const { email, password } = req.body;
  try {
    const user = await User.findOne({ where: { email } });
    if (!user) return res.status(401).json({ message: 'Credenciais inválidas' });

    const valid = await user.validatePassword(password);
    if (!valid) return res.status(401).json({ message: 'Credenciais inválidas' });

    const token = jwt.sign(
      { id: user.id, email: user.email },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRES_IN || '1h' }
    );

    res.json({ token });
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Erro no servidor' });
  }
};
```

---

### `src/middleware/auth.js`

```js
const jwt = require('jsonwebtoken');
const { User } = require('../models');

module.exports = async (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader) return res.status(401).json({ message: 'Token ausente' });

  const [type, token] = authHeader.split(' ');
  if (type !== 'Bearer' || !token) return res.status(401).json({ message: 'Token inválido' });

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findByPk(payload.id);
    if (!user) return res.status(401).json({ message: 'Usuário não encontrado' });

    req.user = user;
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Token inválido ou expirado' });
  }
};
```

---

### `src/routes/users.js`

```js
const express = require('express');
const auth = require('../middleware/auth');
const { User } = require('../models');

const router = express.Router();

router.get('/', auth, async (req, res) => {
  const users = await User.findAll({ attributes: ['id', 'name', 'email', 'createdAt'] });
  res.json({ users });
});

module.exports = router;
```

---
