# Melhorias no Sistema de Banco de Dados - Tech Store

## 1. Estrutura Atual do Banco de Dados

O projeto utiliza **Supabase** como backend, com as seguintes tabelas principais:

### Tabelas Existentes:
- `products` - Produtos da loja
- `categories` - Categorias de produtos
- `cart_items` - Itens no carrinho
- `orders` - Pedidos realizados
- `order_items` - Itens dos pedidos
- `addresses` - Endereços dos usuários
- `auth.users` - Usuários (gerenciado pelo Supabase Auth)

## 2. Melhorias Propostas

### 2.1. Otimização de Queries

**Problema Atual:** Queries espalhadas pelo código frontend, sem otimização.

**Solução Implementada:**
- Centralização de queries em módulos específicos
- Uso de `select` específico para reduzir transferência de dados
- Implementação de paginação para listas grandes
- Cache de resultados frequentes

### 2.2. Índices Recomendados

Para melhorar a performance das consultas mais frequentes:

```sql
-- Índice para busca de produtos por categoria
CREATE INDEX idx_products_category_id ON products(category_id);

-- Índice para busca de produtos por nome (busca textual)
CREATE INDEX idx_products_name_gin ON products USING gin(to_tsvector('portuguese', name));

-- Índice para busca de produtos por descrição
CREATE INDEX idx_products_description_gin ON products USING gin(to_tsvector('portuguese', description));

-- Índice para carrinho por usuário
CREATE INDEX idx_cart_items_user_id ON cart_items(user_id);

-- Índice para pedidos por usuário e data
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Índice para itens de pedido
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

### 2.3. Views para Consultas Complexas

**View para Produtos com Categoria:**
```sql
CREATE VIEW products_with_category AS
SELECT 
    p.*,
    c.name as category_name,
    c.slug as category_slug
FROM products p
LEFT JOIN categories c ON p.category_id = c.id;
```

**View para Carrinho Completo:**
```sql
CREATE VIEW cart_items_detailed AS
SELECT 
    ci.id,
    ci.user_id,
    ci.quantity,
    ci.created_at,
    p.id as product_id,
    p.name as product_name,
    p.price,
    p.image_url,
    p.stock_quantity,
    (ci.quantity * p.price) as total_price
FROM cart_items ci
JOIN products p ON ci.product_id = p.id;
```

### 2.4. Funções do Banco de Dados

**Função para Calcular Total do Carrinho:**
```sql
CREATE OR REPLACE FUNCTION calculate_cart_total(user_uuid UUID)
RETURNS DECIMAL AS $$
BEGIN
    RETURN (
        SELECT COALESCE(SUM(ci.quantity * p.price), 0)
        FROM cart_items ci
        JOIN products p ON ci.product_id = p.id
        WHERE ci.user_id = user_uuid
    );
END;
$$ LANGUAGE plpgsql;
```

**Função para Verificar Estoque:**
```sql
CREATE OR REPLACE FUNCTION check_stock_availability(product_uuid UUID, requested_quantity INTEGER)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN (
        SELECT stock_quantity >= requested_quantity
        FROM products
        WHERE id = product_uuid
    );
END;
$$ LANGUAGE plpgsql;
```

### 2.5. Triggers para Integridade de Dados

**Trigger para Atualizar Estoque:**
```sql
CREATE OR REPLACE FUNCTION update_stock_on_order()
RETURNS TRIGGER AS $$
BEGIN
    -- Reduzir estoque quando pedido é criado
    UPDATE products 
    SET stock_quantity = stock_quantity - NEW.quantity
    WHERE id = NEW.product_id;
    
    -- Verificar se estoque não ficou negativo
    IF (SELECT stock_quantity FROM products WHERE id = NEW.product_id) < 0 THEN
        RAISE EXCEPTION 'Estoque insuficiente para o produto %', NEW.product_id;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_stock
    AFTER INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION update_stock_on_order();
```

### 2.6. Row Level Security (RLS)

**Políticas de Segurança:**

```sql
-- Habilitar RLS nas tabelas
ALTER TABLE cart_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE addresses ENABLE ROW LEVEL SECURITY;

-- Política para cart_items - usuários só veem seus próprios itens
CREATE POLICY "Users can view own cart items" ON cart_items
    FOR ALL USING (auth.uid() = user_id);

-- Política para orders - usuários só veem seus próprios pedidos
CREATE POLICY "Users can view own orders" ON orders
    FOR ALL USING (auth.uid() = user_id);

-- Política para addresses - usuários só veem seus próprios endereços
CREATE POLICY "Users can manage own addresses" ON addresses
    FOR ALL USING (auth.uid() = user_id);
```

## 3. Estrutura de Dados Aprimorada

### 3.1. Campos Adicionais Recomendados

**Tabela Products:**
```sql
ALTER TABLE products ADD COLUMN IF NOT EXISTS rating DECIMAL(2,1) DEFAULT 0;
ALTER TABLE products ADD COLUMN IF NOT EXISTS review_count INTEGER DEFAULT 0;
ALTER TABLE products ADD COLUMN IF NOT EXISTS is_featured BOOLEAN DEFAULT false;
ALTER TABLE products ADD COLUMN IF NOT EXISTS is_new BOOLEAN DEFAULT false;
ALTER TABLE products ADD COLUMN IF NOT EXISTS meta_title VARCHAR(255);
ALTER TABLE products ADD COLUMN IF NOT EXISTS meta_description TEXT;
ALTER TABLE products ADD COLUMN IF NOT EXISTS slug VARCHAR(255) UNIQUE;
```

**Tabela Categories:**
```sql
ALTER TABLE categories ADD COLUMN IF NOT EXISTS description TEXT;
ALTER TABLE categories ADD COLUMN IF NOT EXISTS image_url VARCHAR(500);
ALTER TABLE categories ADD COLUMN IF NOT EXISTS is_active BOOLEAN DEFAULT true;
ALTER TABLE categories ADD COLUMN IF NOT EXISTS sort_order INTEGER DEFAULT 0;
```

### 3.2. Novas Tabelas Recomendadas

**Tabela de Reviews:**
```sql
CREATE TABLE IF NOT EXISTS product_reviews (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    title VARCHAR(255),
    comment TEXT,
    is_verified_purchase BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**Tabela de Cupons de Desconto:**
```sql
CREATE TABLE IF NOT EXISTS discount_coupons (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    discount_type VARCHAR(20) CHECK (discount_type IN ('percentage', 'fixed')),
    discount_value DECIMAL(10,2),
    minimum_order_value DECIMAL(10,2) DEFAULT 0,
    max_uses INTEGER,
    current_uses INTEGER DEFAULT 0,
    valid_from TIMESTAMP WITH TIME ZONE,
    valid_until TIMESTAMP WITH TIME ZONE,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

## 4. API Centralizada

### 4.1. Módulo de API (api.js)

Criação de um módulo centralizado para todas as interações com o Supabase:

```javascript
class DatabaseAPI {
    // Produtos
    async getProducts(filters = {}) { /* ... */ }
    async getProductById(id) { /* ... */ }
    async searchProducts(query) { /* ... */ }
    
    // Carrinho
    async getCartItems(userId) { /* ... */ }
    async addToCart(userId, productId, quantity) { /* ... */ }
    async updateCartItem(itemId, quantity) { /* ... */ }
    
    // Pedidos
    async createOrder(orderData) { /* ... */ }
    async getOrderHistory(userId) { /* ... */ }
    
    // Usuário
    async getUserAddresses(userId) { /* ... */ }
    async saveAddress(userId, addressData) { /* ... */ }
}
```

## 5. Monitoramento e Analytics

### 5.1. Logs de Atividade

```sql
CREATE TABLE IF NOT EXISTS activity_logs (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id),
    action VARCHAR(100),
    resource_type VARCHAR(50),
    resource_id UUID,
    metadata JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### 5.2. Métricas de Performance

- Tempo de resposta das queries
- Produtos mais visualizados
- Taxa de conversão do carrinho
- Produtos mais vendidos

## 6. Backup e Recuperação

### 6.1. Estratégia de Backup

- Backup automático diário do Supabase
- Backup semanal completo
- Retenção de 30 dias para backups diários
- Retenção de 12 semanas para backups semanais

### 6.2. Plano de Recuperação

- RTO (Recovery Time Objective): 4 horas
- RPO (Recovery Point Objective): 24 horas
- Testes de recuperação mensais

## 7. Implementação das Melhorias

As melhorias propostas devem ser implementadas gradualmente:

1. **Fase 1:** Índices e otimização de queries existentes
2. **Fase 2:** Views e funções do banco
3. **Fase 3:** RLS e políticas de segurança
4. **Fase 4:** Novas tabelas e funcionalidades
5. **Fase 5:** Monitoramento e analytics

## 8. Considerações de Segurança

- Todas as queries são executadas através do Supabase RLS
- Validação de dados no frontend e backend
- Sanitização de inputs para prevenir SQL injection
- Auditoria de acessos e modificações
- Criptografia de dados sensíveis

## 9. Performance e Escalabilidade

- Uso de CDN para imagens de produtos
- Cache de queries frequentes
- Paginação para listas grandes
- Lazy loading de imagens
- Compressão de dados transferidos

## 10. Conclusão

As melhorias propostas transformarão o banco de dados em uma solução robusta, segura e escalável, adequada para um e-commerce profissional. A implementação gradual permitirá manter a estabilidade do sistema enquanto adiciona novas funcionalidades.
