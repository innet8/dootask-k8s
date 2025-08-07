# Dootask K8s Argo Workflows 部署指南

## 概述
这个WorkflowTemplate基于提供的Jenkinsfile转换而来，使用ServiceAccount和RBAC进行权限控制，实现了Dootask K8s项目的安全自动化部署流程。

## 主要改进
- ✅ 使用ServiceAccount替代kube-config Secret
- ✅ 通过RBAC精确控制权限
- ✅ 使用emptyDir替代PVC，简化存储配置
- ✅ 遵循Kubernetes安全最佳实践

## 前置条件

### 1. 安装Argo Workflows
```bash
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.4.4/install.yaml
```

### 2. 确保必要的命名空间存在
```bash
# 创建共享Nginx配置的命名空间
kubectl create namespace dootask-saas-share --dry-run=client -o yaml | kubectl apply -f -

# 创建nginx-share-config ConfigMap（如果不存在）
kubectl create configmap nginx-share-config -n dootask-saas-share --dry-run=client -o yaml | kubectl apply -f -
```

## 部署步骤

### 1. 应用完整的WorkflowTemplate（包含RBAC配置）
```bash
kubectl apply -f dootask-workflow-template.yaml
```

这个命令会创建：
- WorkflowTemplate: `dootask-k8s-deploy`
- ServiceAccount: `dootask-deployer`
- ClusterRole: `dootask-deployer`
- ClusterRoleBinding: `dootask-deployer`

### 2. 验证RBAC配置
```bash
# 检查ServiceAccount
kubectl get sa dootask-deployer -n argo

# 检查ClusterRole
kubectl get clusterrole dootask-deployer

# 检查ClusterRoleBinding
kubectl get clusterrolebinding dootask-deployer

# 测试权限
kubectl auth can-i create deployments --as=system:serviceaccount:argo:dootask-deployer -n dootask-test
```

### 3. 运行工作流
```bash
# 使用默认参数运行
argo submit dootask-workflow-template.yaml -n argo

# 自定义参数运行
argo submit dootask-workflow-template.yaml -n argo \
  -p app-id=my-app \
  -p app-key=base64:your-app-key \
  -p db-password=your-db-password \
  -p db-root-password=your-root-password \
  -p namespace=dootask-my-app \
  -p tag=v1.0.0
```

## RBAC权限说明

### ServiceAccount权限范围
创建的`dootask-deployer` ServiceAccount具有以下权限：

#### 命名空间管理
- 创建、查看、修补命名空间

#### 基础资源权限
- Pod、Service、ConfigMap、Secret、PVC的完全管理权限

#### 应用资源权限
- Deployment、StatefulSet、ReplicaSet的完全管理权限
- Job的完全管理权限
- Ingress的完全管理权限

#### 安全考虑
- 使用ClusterRole但限制在特定操作范围内
- 避免使用过于宽泛的权限
- 可根据实际需求进一步细化权限

## 参数说明

| 参数名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| git-url | https://github.com/innet8/dootask-k8s.git | Git仓库地址 | 否 |
| git-branch | main | Git分支 | 否 |
| tag | pro | 镜像标签 | 否 |
| db-password | "" | 数据库密码 | 是 |
| db-root-password | "" | 数据库root密码 | 是 |
| app-key | "" | 应用密钥 | 是 |
| app-id | "" | 应用ID（用于域名和配置） | 是 |
| namespace | dootask-test | Kubernetes部署命名空间 | 否 |

## 工作流程详解

### 1. print-parameters
- 打印所有部署参数，便于调试和确认配置

### 2. git-checkout
- 使用Node.js Alpine镜像克隆代码
- 切换到指定分支
- 将代码存储到emptyDir共享工作空间

### 3. deploy-to-dev
- 通过ServiceAccount权限创建目标命名空间
- 使用envsubst替换config.yaml中的环境变量并应用
- 按顺序应用其他YAML文件
- 等待MariaDB Pod就绪（最多10分钟）
- 应用初始化作业
- 复制TLS证书（如果存在）
- 应用Ingress配置

### 4. update-nginx-share-config
- 通过ServiceAccount权限更新nginx-share-config ConfigMap
- 为新的应用实例添加Nginx代理配置

## 安全优势

### 相比kube-config Secret的优势：
1. **权限最小化** - 只授予必要的权限，不是集群管理员权限
2. **审计友好** - 所有操作都有明确的身份标识
3. **自动轮换** - ServiceAccount token自动管理和轮换
4. **命名空间隔离** - 可以限制在特定命名空间内操作
5. **易于管理** - 通过Kubernetes原生RBAC管理权限

### 安全最佳实践：
1. **定期审查权限** - 检查是否有不必要的权限
2. **监控操作** - 启用审计日志监控ServiceAccount操作
3. **权限分离** - 不同环境使用不同的ServiceAccount
4. **最小权限原则** - 只授予完成任务所需的最小权限

## 监控和调试

### 查看工作流状态
```bash
# 查看工作流列表
argo list -n argo

# 查看特定工作流状态
argo get <workflow-name> -n argo

# 查看工作流日志
argo logs <workflow-name> -n argo -f
```

### 权限问题调试
```bash
# 检查ServiceAccount权限
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:argo:dootask-deployer -n <namespace>

# 查看详细的RBAC信息
kubectl describe clusterrolebinding dootask-deployer
```

## 故障排除

### 常见问题

1. **权限被拒绝**
   ```bash
   # 检查具体权限
   kubectl auth can-i create pods --as=system:serviceaccount:argo:dootask-deployer -n dootask-test
   
   # 如果权限不足，可能需要调整ClusterRole
   kubectl edit clusterrole dootask-deployer
   ```

2. **ServiceAccount不存在**
   ```bash
   # 重新应用RBAC配置
   kubectl apply -f dootask-workflow-template.yaml
   ```

3. **命名空间创建失败**
   ```bash
   # 检查命名空间创建权限
   kubectl auth can-i create namespaces --as=system:serviceaccount:argo:dootask-deployer
   ```

## 自定义权限

如需调整权限，可以修改ClusterRole定义：

```yaml
# 添加新的权限规则
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
```

## 环境特定配置

对于不同环境，建议创建不同的ServiceAccount：

```bash
# 生产环境
kubectl create sa dootask-deployer-prod -n argo

# 测试环境  
kubectl create sa dootask-deployer-test -n argo
```

然后在WorkflowTemplate中指定相应的ServiceAccount。