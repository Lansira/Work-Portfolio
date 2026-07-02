"""
智能客服助手 - 完善版
基于原始导入包实现完整功能
"""

import json
import os
import re
from datetime import datetime
from langchain.agents import create_agent
from langchain.tools import tool
from langchain.agents.middleware import before_model, after_model, dynamic_prompt
from langchain.chat_models import init_chat_model
from langchain.messages import HumanMessage
from langchain_openai import ChatOpenAI

# ==================== 数据加载 ====================

def load_data():
    """加载数据文件"""
    paths = [
        '../data/simple_data.json',
        'data/simple_data.json',
        os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), 'data', 'simple_data.json'),
        'simple_data.json'  # 当前目录
    ]

    for path in paths:
        try:
            with open(path, 'r', encoding='utf-8') as f:
                data = json.load(f)
                print(f"✓ 从 {path} 加载数据成功")
                return data
        except FileNotFoundError:
            continue
        except json.JSONDecodeError as e:
            print(f"✗ 文件 {path} 格式错误: {e}")
            continue

    print("警告：找不到数据文件，使用示例数据")
    return create_sample_data()

def create_sample_data():
    """创建示例数据"""
    return {
        "users": {
            "vip001": {
                "name": "张经理",
                "phone": "138-1234-5678",
                "type": "vip",
                "level": 3,
                "email": "zhang@company.com",
                "join_date": "2022-01-15",
                "address": "北京市朝阳区"
            },
            "normal001": {
                "name": "小李",
                "phone": "139-8765-4321",
                "type": "normal",
                "level": 1,
                "email": "li@example.com",
                "join_date": "2023-05-20",
                "address": "上海市浦东新区"
            },
            "normal002": {
                "name": "小王",
                "phone": "137-9999-8888",
                "type": "normal",
                "level": 2,
                "email": "wang@example.com",
                "join_date": "2023-08-10"
            }
        },
        "orders": {
            "ORD001": {
                "user_id": "vip001",
                "items": "iPhone 15",
                "status": "已发货",
                "date": "2024-03-01",
                "total_amount": 7999.0,
                "shipping_address": "北京市朝阳区",
                "payment_method": "支付宝"
            },
            "ORD002": {
                "user_id": "normal001",
                "items": "充电器",
                "status": "配送中",
                "date": "2024-03-10",
                "total_amount": 199.0,
                "shipping_address": "上海市浦东新区",
                "payment_method": "微信支付"
            },
            "ORD003": {
                "user_id": "normal002",
                "items": "耳机",
                "status": "已送达",
                "date": "2024-03-05",
                "total_amount": 299.0,
                "shipping_address": "广州市天河区",
                "payment_method": "支付宝"
            }
        },
        "faq": [
            {
                "question": "退货政策是什么？",
                "answer": "7天内无理由退货，商品需要保持完好。",
                "category": "售后",
                "keywords": ["退货", "退款", "退换货"]
            },
            {
                "question": "客服电话是多少？",
                "answer": "客服热线是400-888-8888，工作时间9:00-18:00。",
                "category": "联系",
                "keywords": ["电话", "客服", "联系", "热线"]
            },
            {
                "question": "配送时间多久？",
                "answer": "一般地区3-5工作日送达，偏远地区5-7工作日。",
                "category": "物流",
                "keywords": ["配送", "发货", "快递", "物流"]
            }
        ]
    }

# 加载数据
data = load_data()
users = (data or {}).get('users', {})
orders = (data or {}).get('orders', {})
faq_list = (data or {}).get('faq', [])

# ==================== 工具函数 ====================

@tool
def get_user_type(user_id: str) -> str:
    """
    获取用户类型

    Args:
        user_id: 用户ID

    Returns:
        用户类型信息
    """
    if user_id not in users:
        return f"用户 {user_id} 不存在"

    user = users[user_id]
    name = user.get('name', '未知用户')
    phone = user.get('phone', '无联系电话')

    # 处理用户类型
    user_type = user.get('type', 'normal')
    if user_type == 'vip':
        type_str = "VIP客户"
    elif user_type == 'svip':
        type_str = "超级VIP客户"
    else:
        type_str = "普通客户" # 获取额外信息
    extra_info = []
    if 'level' in user:
        extra_info.append(f"等级：Lv{user['level']}")
    if 'join_date' in user:
        join_date = user['join_date']
        try:
            join_dt = datetime.strptime(join_date, "%Y-%m-%d")
            days = (datetime.now() - join_dt).days
            extra_info.append(f"会员时长：{days}天")
        except:
            extra_info.append(f"加入日期：{join_date}")

    base_info = f"{name}是{type_str}，联系电话：{phone}"

    if extra_info:
        return f"{base_info}（{', '.join(extra_info)}）"
    return base_info

@tool
def check_order_status(order_id: str) -> str:
    """
    查询订单状态

    Args:
        order_id: 订单ID

    Returns:
        订单状态信息
    """
    # 处理大小写不敏感
    order_id = order_id.upper()

    if order_id not in orders:
        return f"订单 {order_id} 不存在"

    order = orders[order_id]
    user_id = order.get('user_id')

    # 获取客户信息
    customer_name = "未知客户"
    customer_phone = ""
    if user_id in users:
        customer_name = users[user_id].get('name', '未知客户')
        customer_phone = users[user_id].get('phone', '')

    # 构建订单信息
    order_info = [
        f"📦 订单号：{order_id}",
        f"👤 客户：{customer_name}",
        f"📞 电话：{customer_phone if customer_phone else '无'}",
        f"🛒 商品：{order.get('items', '无商品信息')}",
        f"📊 状态：{order.get('status', '未知状态')}",
        f"📅 日期：{order.get('date', '未知日期')}"
    ]

    # 根据状态添加建议
    status = order.get('status', '')
    if status == '待支付':
        order_info.append("\n 温馨提示：订单尚未支付，请及时完成支付")
    elif status == '处理中':
        order_info.append("\n 温馨提示：订单正在处理中，预计1-2个工作日内发货")
    elif status == '已发货':
        order_info.append("\n 温馨提示：商品已在运输途中，请注意查收")
    elif status == '配送中':
        order_info.append("\n 温馨提示：商品正在配送中，请注意查收")
    elif status == '已送达':
        order_info.append("\n 温馨提示：商品已送达，如有问题请及时联系客服")

    return "\n".join(order_info)


@tool
def answer_faq(question: str) -> str:
    """
    回答常见问题

    Args:
        question: 用户的问题

    Returns:
      根据关键词匹配返回相应的答案
    """
    question_lower = question.lower()

    # 先在FAQ列表中查找
    for faq in faq_list:
        faq_question = faq.get('question', '').lower()
        faq_keywords = faq.get('keywords', [])

        # 检查问题是否匹配FAQ问题或关键词
        if faq_question in question_lower:
            return faq.get('answer', '请拨打客服热线400-888-8888')

        # 检查关键词是否匹配
        for keyword in faq_keywords:
            if keyword in question_lower:
                return faq.get('answer', '请拨打客服热线400-888-8888')

    # 默认回答
    return "抱歉，我没有找到相关问题的答案。请拨打客服热线400-888-8888获取更多帮助。"

@tool
def get_user_orders(user_id: str) -> str:
    """
    获取用户的所有订单

    Args:
        user_id: 用户ID

    Returns:
        用户的订单列表
    """
    if user_id not in users:
        return f"用户 {user_id} 不存在"

    user_orders = []
    for order_id, order in orders.items():
        if order.get('user_id') == user_id:
            user_orders.append((order_id, order))

    if not user_orders:
        return f"{users[user_id].get('name')} 暂无订单记录"
    # 按日期排序（最近在前）
    user_orders.sort(key=lambda x: x[1].get('date', ''), reverse=True)

    user_name = users[user_id].get('name')
    result = [f"👤 {user_name} 的订单记录（共{len(user_orders)}单）："]

    for order_id, order in user_orders:
        items = order.get('items', '')
        status = order.get('status', '')
        date = order.get('date', '')

        # 状态图标
        status_icons = {
            '待支付': '⏳',
            '处理中': '🔄',
            '已发货': '🚚',
            '配送中': '🚚',
            '已送达': '✅',
            '已取消': '❌'
        }
        status_icon = status_icons.get(status, '📄')

        result.append(f"\n{status_icon} {order_id}")
        result.append(f"   商品：{items[:30]}{'...' if len(items) > 30 else ''}")
        result.append(f"   状态：{status}")
        result.append(f"   日期：{date}")

    return "\n".join(result)

# ==================== 中间件实现 ====================

@before_model
def save_user_question(state, runtime):
    """
    保存用户问题到日志文件
    """
    try:
        # 获取用户ID
        user_id = "unknown"
        if runtime and runtime.context:
            user_id = runtime.context.get('user_id', 'unknown')

        # 获取最后一条用户消息
        messages = state.get('messages', [])
        if messages:
            # 查找最后一条HumanMessage
            last_human_msg = None
            for msg in reversed(messages):
                if isinstance(msg, HumanMessage):
                    last_human_msg = msg
                    break

            if last_human_msg:
                content = last_human_msg.content if hasattr(last_human_msg, 'content') else str(last_human_msg)

                # 写入日志
                timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                log_entry = f"{timestamp} - {user_id} - {content}\n"

                with open('question_log.txt', 'a', encoding='utf-8') as f:
                        f.write(log_entry)
    except Exception as e:
        print(f"写入日志失败：{e}")

    return state

@after_model
def hide_phone_numbers(state, runtime):
    """
    隐藏回复中的手机号
    """
    try:
        messages = state.get('messages', [])
        if messages:
            last_message = messages[-1]
            if hasattr(last_message, 'content'):
                content = last_message.content

                # 正则表达式匹配11位手机号
                def replace_phone(match):
                    phone_str = match.group()
                    # 提取所有数字
                    digits = ''.join(filter(str.isdigit, phone_str))
                    if len(digits) == 11:
                        # 格式化为 138-****-5678
                        return f"{digits[:3]}-****-{digits[-4:]}"
                    return phone_str

                # 匹配各种格式的11位手机号
                pattern = r'1[3-9]\d[\s\-]?\d{4}[\s\-]?\d{4}'
                content = re.sub(pattern, replace_phone, content)

                # 更新消息内容
                last_message.content = content
    except Exception as e:
        print(f"隐藏手机号失败：{e}")

    return state

@dynamic_prompt
def create_prompt(request):
    """
    根据用户类型创建不同的提示词

    注意：这里返回字符串而不是字典
    """
    # 获取用户ID
    user_id = "unknown"
    if request.runtime and request.runtime.context:
        user_id = request.runtime.context.get('user_id', 'unknown')

    # 获取用户类型
    user_type = "normal"
    user_name = "客户"
    if user_id in users:
        user_info = users[user_id]
        user_type = user_info.get('type', 'normal')
        user_name = user_info.get('name', '客户')

    # 基础系统提示
    base_prompt = """你是一个智能客服助手，专门处理客户咨询。你有以下
工具可用：
1. get_user_type: 查询用户类型和信息
2. check_order_status: 查询订单状态
3. answer_faq: 回答常见问题
4. get_user_orders: 获取用户的所有订单

请根据用户的问题选择合适的工具，并给出清晰、准确、有用的回答。"""

    # 根据用户类型定制提示
    if user_type == 'vip':
        return f"""{base_prompt}

你现在正在为VIP客户【{user_name}】提供服务，请遵守以下VIP服务准则：
1. 使用尊称"您"，体现最高尊重
2. 优先处理VIP客户的请求
3. 提供更详细的解答和个性化建议
4. 可以适当超出标准流程提供帮助
5. 态度要特别热情、专业、耐心
6. 记得称呼客户为"张经理"或其他尊称
7. 对于VIP客户的问题要优先、快速响应

请记住：这是VIP客户，需要提供最高级别的服务。"""
    else:
        return f"""{base_prompt}

你现在正在为客户【{user_name}】提供服务，请遵守以下服务准则：
1. 态度友好、专业、耐心
2. 回答要准确、清晰、有用
3. 对于复杂问题，建议联系人工客服
4. 使用标准礼貌用语
5. 提供力所能及的帮助

请提供优质的标准服务。"""

# ==================== 主程序 ====================

class CustomerServiceAgent:
    """智能客服助手类"""
    def __init__(self, use_local_model=True):
        """
        初始化智能体
        """
        self.use_local_model = use_local_model
        if use_local_model:
            try:
                # 尝试创建ChatOpenAI模型（连接到本地Ollama）
                self.model = ChatOpenAI(
                    model="qwen2.5:7b",  # 或 "qwen3", "llama2", "mistral" 等
                    base_url="http://localhost:11434/v1/",
                    api_key="ollama",
                    temperature=0.3,
                    max_tokens=1000
                )
                print("✓ 已连接到本地语言模型")

                # 定义工具
                self.tools = [get_user_type, check_order_status, answer_faq, get_user_orders]

                # 定义中间件
                self.middleware = [save_user_question, hide_phone_numbers, create_prompt]

                # 创建智能体
                self.agent = create_agent(
                    model=self.model,
                    tools=self.tools,
                    middleware=self.middleware
                )
                print("✓ 智能体创建成功")

            except Exception as e:
                print(f"⚠ 无法连接本地模型或创建智能体失败：{e}")
                print("⚠ 使用模拟模式")
                self.model = None
                self.agent = None
                self.use_local_model = False
        else:
            self.model = None
            self.agent = None
            self.use_local_model = False
    def _simulate_response(self, user_id: str, message: str) -> str:
        """模拟AI响应（当模型不可用时）"""
        message_lower = message.lower()
        # 检查是否查询用户信息
        if any(keyword in message_lower for keyword in['我是谁', '我的信息', '用户信息', '我是', '用户类型']):
            return get_user_type.invoke(user_id)

        # 检查是否查询订单
        if any(keyword in message_lower for keyword in['订单', 'ord', '下单', '购买', '订单状态', '查看订单']):
            # 尝试提取订单号
            order_match = re.search(r'ORD\d+', message.upper())
            if order_match:
                return check_order_status.invoke(order_match.group())
            else:
                # 查询用户所有订单
                return get_user_orders.invoke(user_id)

        # 检查常见问题
        if any(keyword in message_lower for keyword in['退货', '退款', '配送', '发货', '电话', '客服', '支付', '会员', '政策', '常见问题']):
            return answer_faq.invoke(message)

        # 问候语
        if any(keyword in message_lower for keyword in ['你好', '您好', 'hi', 'hello', '嗨']):
            user_name = users.get(user_id, {}).get('name', '客户')
            return f"{user_name}，您好！我是智能客服助手，很高兴为您服务。"

        # 帮助信息
        if any(keyword in message_lower for keyword in ['帮助', '功能', '能做什么', '可以做什么']):
            return """我可以帮助您：
1. 查询用户信息 - 例如："我是谁" 或 "查询我的信息"
2. 查询订单状态 - 例如："查询ORD001状态" 或 "查看我的订单"
3. 回答常见问题 - 例如："退货政策是什么" 或 "客服电话多少"
4. 查看所有订单 - 例如："查看我的所有订单"

请告诉我您需要什么帮助？"""

        # 默认回复
        user_name = users.get(user_id, {}).get('name', '客户')
        return f"{user_name}，您好！我可以帮助您查询用户信息、订单状态、回答常见问题等。请告诉我您需要什么帮助？"
    def chat(self, user_id: str, message: str) -> str:
        """
        与用户对话

        Args:
            user_id: 用户ID
            message: 用户消息

        Returns:
            AI助手的回复
        """
        # 检查用户是否存在
        if user_id not in users and user_id not in ['unknown', 'guest']:
            return f"用户 {user_id} 不存在，请使用有效的用户ID"

        # 记录开始时间
        start_time = datetime.now()

        try:
            if self.agent is None:
                # 使用模拟模式
                print(f"🤖 模拟模式处理：用户[{user_id}] - {message[:50]}...")
                response = self._simulate_response(user_id, message)
            else:
                # 使用AI模型
                print(f"🤖 AI处理中：用户[{user_id}] - {message[:50]}...")

                # 创建消息
                human_message = HumanMessage(content=message)

                # 准备上下文
                context = {"user_id": user_id}

                # 调用智能体
                try:
                    response = self.agent.invoke(
                        input={"messages": [human_message]},
                        config={"context": context}
                    )
                except Exception as e:
                    print(f"⚠ AI调用失败，降级到模拟模式：{e}")
                    response = self._simulate_response(user_id, message)

                # 提取回复内容
                if hasattr(response, 'content'):
                    response = response.content
                elif isinstance(response, dict) and 'content' in response:
                    response = response['content']
                elif isinstance(response, dict) and 'output' in response:
                    response = response['output']
                else:
                    response = str(response)

            # 计算响应时间
            response_time = (datetime.now() - start_time).total_seconds()

            # 记录对话（简化的日志）
            self._log_conversation(user_id, message, response, response_time)

            return response

        except Exception as e:
            error_msg = f"处理请求时出错：{str(e)}"
            print(f"❌ {error_msg}")

            # 降级处理：使用模拟响应
            try:
                return self._simulate_response(user_id, message)
            except:
                return f"抱歉，系统暂时无法处理您的请求。请稍后再试。"

    def _log_conversation(self, user_id: str, question: str, response: str, response_time: float):
        """记录对话到日志文件"""
        try:
            timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            log_entry = f"[{timestamp}] 用户[{user_id}] 问：{question}\n"
            log_entry += f"[{timestamp}] 客服答：{response[:100]}{'...' if len(response) > 100 else ''}\n"
            log_entry += f"[{timestamp}] 响应时间：{response_time:.2f}秒\n"
            log_entry += "-" * 50 + "\n"
            with open('conversation_log.txt', 'a', encoding='utf-8') as f:
                f.write(log_entry)

        except Exception as e:
            print(f"记录对话日志失败：{e}")

# ==================== 测试运行 ====================
def test_tools():
    """测试工具函数"""
    print("🧪 测试工具函数...")
    print("=" * 50)

    # 测试get_user_type
    print("1. 测试 get_user_type:")
    print(get_user_type.invoke("vip001"))
    print(get_user_type.invoke("normal001"))
    print(get_user_type.invoke("unknown_user"))

    print("\n2. 测试 check_order_status:")
    print(check_order_status.invoke("ORD001"))
    print(check_order_status.invoke("ORD002"))
    print(check_order_status.invoke("ORD999"))

    print("\n3. 测试 answer_faq:")
    print("Q: 退货政策")
    print("A:", answer_faq.invoke("退货政策"))
    print("\nQ: 客服电话多少")
    print("A:", answer_faq.invoke("客服电话多少"))
    print("\nQ: 什么时候发货")
    print("A:", answer_faq.invoke("什么时候发货"))

    print("\n4. 测试 get_user_orders:")
    print(get_user_orders.invoke("vip001"))

    print("\n" + "=" * 50)
def test_middleware_function():
    """测试中间件功能（不依赖实际state/runtime）"""
    print("🧪 测试中间件功能...")
    print("=" * 50)

    # 测试手机号隐藏功能
    test_cases = [
        "我的电话是138-1234-5678",
        "另一个电话是13987654321",
        "客服电话400-888-8888",
        "紧急电话138 1234 5678"
    ]

    print("测试手机号隐藏功能：")
    for test_text in test_cases:
        print(f"\n原始文本：{test_text}")

        # 直接测试隐藏逻辑
        def hide_phone_numbers_direct(text):
            def replace_phone(match):
                phone_str = match.group()
                # 提取所有数字
                digits = ''.join(filter(str.isdigit, phone_str))
                if len(digits) == 11:
                    # 格式化为 138-****-5678
                    return f"{digits[:3]}-****-{digits[-4:]}"
                return phone_str

            # 匹配各种格式的11位手机号
            pattern = r'1[3-9]\d[\s\-]?\d{4}[\s\-]?\d{4}'
            return re.sub(pattern, replace_phone, text)

        result = hide_phone_numbers_direct(test_text)
        print(f"处理后：{result}")

    print("\n✓ 中间件功能测试完成")

def main():
    """主函数 - 演示测试"""
    print("=" * 60)
    print("🤖 智能客服助手演示")
    print("=" * 60)
    print("📊 数据统计：")
    print(f"  • 用户数量：{len(users)}")
    print(f"  • 订单数量：{len(orders)}")
    print(f"  • FAQ数量：{len(faq_list)}")

    print("\n👥 可用用户：")
    for user_id, user_info in users.items():
        user_type = user_info.get('type', 'normal')
        type_map = {'vip': 'VIP', 'svip': '超级VIP', 'normal': '普通'}
        print(f"  • {user_id}: {user_info['name']} ({type_map.get(user_type, '普通')})")

    print("\n📦 可用订单：")
    for order_id in orders.keys():
        print(f"  • {order_id}")

    print("\n" + "=" * 60)

    # 测试工具函数
    test_choice = input("是否先测试工具函数？(y/n): ").lower()
    if test_choice == 'y':
        test_tools()

    # 测试中间件
    test_choice = input("是否测试中间件功能？(y/n): ").lower()
    if test_choice == 'y':
        test_middleware_function()

    print("\n" + "=" * 60)
    print("🚀 创建智能客服助手...")

    # 选择模式
    use_ai = input("是否使用AI模型？(需要运行Ollama)(y/n): ").lower() == 'y'

    # 创建智能体
    agent = CustomerServiceAgent(use_local_model=use_ai)
    print("\n" + "=" * 60)
    print("💬 开始对话测试（输入 'quit' 退出）")
    print("=" * 60)

    # 选择用户
    print("\n选择用户：")
    for i, user_id in enumerate(users.keys(), 1):
        user_info = users[user_id]
        print(f"  {i}. {user_id} ({user_info['name']})")
    print(f"  {len(users) + 1}. 新用户（测试不存在的用户）")

    try:
        choice = input(f"\n请选择用户 (输入数字或用户ID): ").strip()
        if choice.isdigit():
            choice_num = int(choice)
            if 1 <= choice_num <= len(users):
                user_id = list(users.keys())[choice_num - 1]
            else:
                user_id = "new_user"
        elif choice in users:
            user_id = choice
        else:
            user_id = "vip001"  # 默认用户
    except:
        user_id = "vip001"

    user_name = users.get(user_id, {}).get('name', '新用户')
    print(f"\n👤 当前用户：{user_name} ({user_id})")

    # 对话循环
    while True:
        try:
            message = input(f"\n[{user_name}] > ").strip()

            if message.lower() in ['quit', 'exit', '退出']:
                print("再见！")
                break
            elif message.lower() == 'help':
                print("\n💡 帮助：")
                print("  您可以问：")
                print("  • '我是谁' - 查看您的信息")
                print("  • '我的订单' - 查看您的所有订单")
                print("  • 'ORD001状态' - 查询特定订单")
                print("  • '退货政策' - 查看退货政策")
                print("  • '客服电话' - 获取联系方式")
                print("  • '支付方式' - 查看支持的支付方式")
                print("  • 'switch' - 切换用户")
                print("  • 'quit' - 退出程序")
                continue
            elif message.lower() == 'switch':
                # 切换用户
                print("\n切换用户：")
                for i, uid in enumerate(users.keys(), 1):
                    user_info = users[uid]
                    print(f"  {i}. {uid} ({user_info['name']})")

                try:
                    choice = input(f"请选择用户 (1-{len(users)}): ")
                    if choice.isdigit():
                        choice_num = int(choice)
                        if 1 <= choice_num <= len(users):
                            user_id = list(users.keys())[choice_num - 1]
                            user_name = users[user_id]['name']
                            print(f"已切换到用户：{user_name}")
                        else:
                            print("无效选择")
                    elif choice in users:
                        user_id = choice
                        user_name = users[user_id]['name']
                        print(f"已切换到用户：{user_name}")
                    else:
                        print("无效输入")
                except:
                    print("无效输入")
                continue

            if not message:
                continue

            # 处理消息
            response = agent.chat(user_id, message)

            # 显示回复
            print(f"\n🤖 客服：")
            print(response)

        except KeyboardInterrupt:
            print("\n\n程序被中断，再见！")
            break
        except Exception as e:
            print(f"❌ 发生错误：{e}")

    # 显示日志

    print("\n" + "=" * 60)
    print("📋 对话日志已保存到文件：")
    print("  • question_log.txt - 用户问题日志")
    print("  • conversation_log.txt - 完整对话日志")

    # 显示最新的用户问题日志
    try:
        with open('question_log.txt', 'r', encoding='utf-8') as f:
            logs = f.readlines()
            if logs:
                print(f"\n最近的用户问题（共{len(logs)}条）：")
                for log in logs[-5:]:  # 显示最后5条
                    print(f"  • {log.strip()}")
    except FileNotFoundError:
        print("暂无用户问题日志")

    print("\n🎉 演示结束！")
# 运行主程序
if __name__ == "__main__":
    main()
