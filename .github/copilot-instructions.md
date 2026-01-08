# NutriMate AI 编码助手指南

你是一名专业的 AI 开发者，正在参与 **NutriMate（营养管家）** —— 临床营养智能管理系统的开发。

## 🏗️ 架构概览

NutriMate 采用 **Monorepo** 结构，基于 **CS + BS 混合架构**，专为医院临床营养科设计。

```
┌─────────────────┐   HTTP/REST   ┌─────────────────┐   gRPC   ┌─────────────────┐
│  Vue 3 前端     │ ───────────▶ │  Go 核心服务    │ ───────▶ │ Python AI 服务  │
│  (Web + Tauri)  │              │   (Gin + GORM)   │          │   (FastAPI)     │
└─────────────────┘              └─────────────────┘          └─────────────────┘
                                        │                            │
                                        ▼                            ▼
                                 PostgreSQL + Redis              Milvus (向量库)
```

### 服务边界

| 服务         | 职责                                | 技术栈            |
| ------------ | ----------------------------------- | ----------------- |
| **前端**     | 单一代码库 → Web + 桌面端 (Win/Mac) | Vue 3 + Tauri 2.0 |
| **核心服务** | 认证、用户、处方管理、医嘱流转      | Go (Gin + GORM)   |
| **AI 服务**  | 大模型接口、RAG 检索、营养推理      | Python (FastAPI)  |

## 📂 项目结构

```
NutriMate/
├── frontend/              # Vue 3 + Tauri 桌面应用
│   ├── src/               # Vue 组件、Pinia 状态管理
│   └── src-tauri/         # Tauri Rust 配置（仅配置，无复杂 Rust 逻辑）
├── backend/
│   ├── core-server/       # Go 核心服务
│   │   ├── cmd/           # 程序入口
│   │   ├── internal/      # Domain、Service、Repository 分层
│   │   └── pkg/           # 公共工具库
│   ├── ai-server/         # Python AI 服务
│   │   ├── app/           # FastAPI 路由
│   │   ├── models/        # 机器学习模型
│   │   └── vector_store/  # Milvus 向量库交互
│   └── proto/             # gRPC 协议定义 (.proto 文件)
├── deploy/                # Docker Compose、K8s 部署配置
└── docs/                  # 项目文档（技术选型方案.md 等）
```

## 💻 技术栈与开发规范

### 前端 (Vue 3 + Tauri)

- **框架**：Vue 3 Composition API，使用 `<script setup lang="ts">`
- **UI 组件库**：**仅使用 Element Plus** —— 针对桌面端高密度 UI、复杂表单和数据表格优化
- **状态管理**：Pinia
- **构建工具**：Vite；包管理器使用 `pnpm`
- **桌面端**：Tauri 2.0 封装同一 SPA（无需单独代码库）

**开发模式**：

- Web 和桌面端代码复用率 100%，不做移动端适配
- 患者列表使用 `el-table`，处方表单使用 `el-form` 并配置严格校验
- API 调用通过 `axios`；TypeScript 接口必须与 Go 结构体 JSON tag 完全对应

### 核心后端 (Go)

- **框架**：Gin + GORM (PostgreSQL)
- **项目布局**：标准 Go 项目结构 (`cmd/`、`internal/`、`pkg/`)
- **认证**：JWT + RBAC 权限控制
- **缓存**：Redis 用于会话管理和热点数据
- **gRPC 客户端**：调用 AI 服务获取智能功能

### AI 后端 (Python)

- **框架**：FastAPI（异步）
- **向量库**：Milvus 用于 RAG 知识检索
- **大模型**：集成 Qwen/ChatGLM 或云端 API（Azure OpenAI、通义千问）
- **gRPC 服务端**：向核心服务暴露接口

## 🛠️ 开发工作流

### 本地运行（单服务）

```bash
# 前端 (Web)
cd frontend && pnpm install && pnpm dev   # http://localhost:5173

# 前端 (桌面端)
pnpm tauri dev   # 需要 Rust 工具链

# Go 核心服务
cd backend/core-server && go run cmd/server/main.go

# Python AI 服务
cd backend/ai-server && pip install -r requirements.txt && uvicorn app.main:app --reload
```

### Docker Compose 全栈启动

```bash
# 启动全部服务（首次需要构建镜像）
cd deploy && docker-compose up -d --build

# 仅启动基础设施（数据库、缓存、向量库）
docker-compose up -d postgres redis milvus

# 查看服务日志
docker-compose logs -f core-server ai-server

# 停止并清理
docker-compose down -v
```

### 运行测试

```bash
# 前端单元测试
cd frontend && pnpm test

# 前端 E2E 测试
pnpm test:e2e

# Go 单元测试
cd backend/core-server && go test ./...

# Go 测试覆盖率
go test -coverprofile=coverage.out ./... && go tool cover -html=coverage.out

# Python 测试
cd backend/ai-server && pytest tests/ -v

# Python 测试覆盖率
pytest --cov=app tests/
```

### gRPC Proto 代码生成

修改 `backend/proto/*.proto` 后，需重新生成 stub 代码：

```bash
# 安装依赖工具（首次）
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
pip install grpcio-tools

# 生成 Go 代码
protoc --proto_path=backend/proto \
  --go_out=backend/core-server/internal/grpc --go_opt=paths=source_relative \
  --go-grpc_out=backend/core-server/internal/grpc --go-grpc_opt=paths=source_relative \
  backend/proto/*.proto

# 生成 Python 代码
python -m grpc_tools.protoc --proto_path=backend/proto \
  --python_out=backend/ai-server/app/grpc \
  --grpc_python_out=backend/ai-server/app/grpc \
  backend/proto/*.proto
```

### 关键集成点

1. **gRPC 变更**：修改 `.proto` 后执行上述生成命令，确保 Go 和 Python 两端同步
2. **数据库迁移**：由 Go 服务通过 GORM auto-migrate 管理
3. **类型契约**：前端 TypeScript 接口必须与 Go 结构体的 JSON tag 精确匹配

## 🐳 Docker 部署

### 服务端口映射

| 服务         | 容器端口 | 主机端口 |
| ------------ | -------- | -------- |
| 前端 (Nginx) | 80       | 8080     |
| Go 核心服务  | 8000     | 8000     |
| AI 服务      | 8001     | 8001     |
| PostgreSQL   | 5432     | 5432     |
| Redis        | 6379     | 6379     |
| Milvus       | 19530    | 19530    |

### 生产部署流程

```bash
# 1. 构建前端生产包
cd frontend && pnpm build

# 2. 构建 Docker 镜像
docker build -t nutrimate/frontend:latest -f deploy/Dockerfile.frontend .
docker build -t nutrimate/core-server:latest -f deploy/Dockerfile.core .
docker build -t nutrimate/ai-server:latest -f deploy/Dockerfile.ai .

# 3. 启动生产环境
cd deploy && docker-compose -f docker-compose.prod.yml up -d

# 4. 查看运行状态
docker-compose ps
```

### 环境变量配置

创建 `deploy/.env` 文件：

```env
# 数据库
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=nutrimate
POSTGRES_USER=nutrimate
POSTGRES_PASSWORD=your_secure_password

# Redis
REDIS_HOST=redis
REDIS_PORT=6379

# AI 服务
OPENAI_API_KEY=sk-xxx
MILVUS_HOST=milvus
MILVUS_PORT=19530

# JWT
JWT_SECRET=your_jwt_secret_key
```

## 📝 代码示例

### Vue 组件模板（处方表单）

```vue
<script setup lang="ts">
import { ref, reactive } from "vue";
import type { FormInstance, FormRules } from "element-plus";
import { createPrescription } from "@/api/prescription";
import type { PrescriptionForm } from "@/types/prescription";

const formRef = ref<FormInstance>();
const form = reactive<PrescriptionForm>({
  patientId: "",
  nutritionType: "enteral", // 肠内营养 | 肠外营养
  calories: 0,
  protein: 0,
  remarks: "",
});

const rules: FormRules = {
  patientId: [{ required: true, message: "请选择患者", trigger: "change" }],
  calories: [
    { required: true, type: "number", min: 100, message: "热量不低于100kcal" },
  ],
  protein: [
    { required: true, type: "number", min: 0, message: "蛋白质不能为负" },
  ],
};

const handleSubmit = async () => {
  await formRef.value?.validate();
  await createPrescription(form);
  ElMessage.success("处方创建成功");
};
</script>

<template>
  <el-form ref="formRef" :model="form" :rules="rules" label-width="100px">
    <el-form-item label="患者" prop="patientId">
      <el-select v-model="form.patientId" placeholder="请选择患者" filterable />
    </el-form-item>
    <el-form-item label="营养类型" prop="nutritionType">
      <el-radio-group v-model="form.nutritionType">
        <el-radio value="enteral">肠内营养</el-radio>
        <el-radio value="parenteral">肠外营养</el-radio>
      </el-radio-group>
    </el-form-item>
    <el-form-item label="热量(kcal)" prop="calories">
      <el-input-number v-model="form.calories" :min="0" :step="100" />
    </el-form-item>
    <el-form-item label="蛋白质(g)" prop="protein">
      <el-input-number v-model="form.protein" :min="0" :precision="1" />
    </el-form-item>
    <el-form-item>
      <el-button type="primary" @click="handleSubmit">提交处方</el-button>
    </el-form-item>
  </el-form>
</template>
```

### Go Handler 示例（Gin）

```go
// internal/handler/prescription.go
package handler

import (
    "net/http"
    "github.com/gin-gonic/gin"
    "nutrimate/internal/service"
    "nutrimate/internal/domain"
)

type PrescriptionHandler struct {
    svc *service.PrescriptionService
}

// CreatePrescription 创建营养处方
// @Summary 创建营养处方
// @Tags 处方管理
// @Accept json
// @Produce json
// @Param body body domain.CreatePrescriptionReq true "处方信息"
// @Success 200 {object} domain.Prescription
// @Router /api/v1/prescriptions [post]
func (h *PrescriptionHandler) CreatePrescription(c *gin.Context) {
    var req domain.CreatePrescriptionReq
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // 获取当前用户ID（从JWT中解析）
    userID := c.GetString("userID")

    prescription, err := h.svc.Create(c.Request.Context(), userID, &req)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, prescription)
}
```

### Go Domain 结构体

```go
// internal/domain/prescription.go
package domain

import "time"

// Prescription 营养处方实体
type Prescription struct {
    ID            string    `json:"id" gorm:"primaryKey"`
    PatientID     string    `json:"patientId" gorm:"index"`
    DoctorID      string    `json:"doctorId"`
    NutritionType string    `json:"nutritionType"` // enteral | parenteral
    Calories      float64   `json:"calories"`
    Protein       float64   `json:"protein"`
    Status        string    `json:"status"` // draft | pending | approved | rejected
    Remarks       string    `json:"remarks"`
    CreatedAt     time.Time `json:"createdAt"`
    UpdatedAt     time.Time `json:"updatedAt"`
}

// CreatePrescriptionReq 创建处方请求
type CreatePrescriptionReq struct {
    PatientID     string  `json:"patientId" binding:"required"`
    NutritionType string  `json:"nutritionType" binding:"required,oneof=enteral parenteral"`
    Calories      float64 `json:"calories" binding:"required,min=100"`
    Protein       float64 `json:"protein" binding:"min=0"`
    Remarks       string  `json:"remarks"`
}
```

### Python FastAPI 示例（AI 服务）

```python
# app/routers/recommendation.py
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from typing import Optional
from app.services.ai_engine import AIEngine
from app.services.vector_store import MilvusClient

router = APIRouter(prefix="/api/v1/recommendations", tags=["AI推荐"])

class RecommendationRequest(BaseModel):
    patient_id: str
    diagnosis: str
    current_nutrition: Optional[dict] = None

class RecommendationResponse(BaseModel):
    recommended_calories: float
    recommended_protein: float
    suggested_formulas: list[str]
    reasoning: str

@router.post("/", response_model=RecommendationResponse)
async def get_nutrition_recommendation(
    req: RecommendationRequest,
    ai_engine: AIEngine = Depends(),
    vector_client: MilvusClient = Depends()
):
    """基于患者诊断和RAG知识库生成营养推荐"""
    # 1. 从向量库检索相关营养指南
    context = await vector_client.search_similar(
        query=req.diagnosis,
        collection="nutrition_guidelines",
        top_k=5
    )

    # 2. 调用大模型生成推荐
    recommendation = await ai_engine.generate_recommendation(
        patient_context=req.dict(),
        rag_context=context
    )

    return recommendation
```

## ⚠️ 项目特定规则

1. **医疗数据安全**：以医疗级合规标准处理患者数据 —— 敏感字段加密、完善审计日志
2. **仅用 Element Plus**：不引入其他 UI 组件库，保持桌面端优先的一致体验
3. **服务通信规范**：前端 → 核心服务走 REST；核心服务 ↔ AI 服务走 gRPC。禁止前端直接调用 AI 服务
4. **双语环境**：代码使用英文；用户界面文案和文档使用中文

## 🧠 AI 功能开发指南

- 开发新功能时，思考如何让 AI 服务 (Python) 增强核心服务 (Go) 的业务逻辑
- 营养推荐流程：核心服务获取患者数据 → gRPC 调用 AI 服务 → AI 查询 Milvus 进行 RAG 检索 → 返回推荐结果
- 领域知识：系统内置 NRS-2002、PG-SGA 等标准营养筛查量表

## 📚 关键文档

- [docs/NutriMate 技术选型方案.md](docs/NutriMate技术选型方案.md) — 技术选型详细说明
- [docs/NutriMate 项目起草方案.md](docs/NutriMate项目起草方案.md) — 产品愿景与功能规划
