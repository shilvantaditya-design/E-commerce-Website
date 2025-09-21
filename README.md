# E-commerce-Website
from fastapi import FastAPI, APIRouter, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from dotenv import load_dotenv
from starlette.middleware.cors import CORSMiddleware
from motor.motor_asyncio import AsyncIOMotorClient
import os
import logging
from pathlib import Path
from pydantic import BaseModel, Field
from typing import List, Optional
import uuid
from datetime import datetime, timezone
import hashlib
import jwt
from enum import Enum

ROOT_DIR = Path(__file__).parent
load_dotenv(ROOT_DIR / '.env')

# MongoDB connection
mongo_url = os.environ['MONGO_URL']
client = AsyncIOMotorClient(mongo_url)
db = client[os.environ['DB_NAME']]

# Create the main app without a prefix
app = FastAPI()

# Create a router with the /api prefix
api_router = APIRouter(prefix="/api")

# Security
security = HTTPBearer()
JWT_SECRET = "your-secret-key-change-in-production"

# Enums
class OrderStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class ProductCategory(str, Enum):
    CLOTHING = "clothing"
    JEWELRY = "jewelry"
    HANDBAGS = "handbags"

# Models
class Product(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    name: str
    description: str
    price: float
    category: ProductCategory
    image_url: str
    stock_quantity: int = 0
    featured: bool = False
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

class ProductCreate(BaseModel):
    name: str
    description: str
    price: float
    category: ProductCategory
    image_url: str
    stock_quantity: int = 0
    featured: bool = False

class ProductUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    price: Optional[float] = None
    category: Optional[ProductCategory] = None
    image_url: Optional[str] = None
    stock_quantity: Optional[int] = None
    featured: Optional[bool] = None

class CartItem(BaseModel):
    product_id: str
    quantity: int
    price: float

class Order(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    customer_name: str
    customer_email: str
    customer_phone: str
    shipping_address: str
    items: List[CartItem]
    total_amount: float
    status: OrderStatus = OrderStatus.PENDING
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

class OrderCreate(BaseModel):
    customer_name: str
    customer_email: str
    customer_phone: str
    shipping_address: str
    items: List[CartItem]
    total_amount: float

class AdminLogin(BaseModel):
    username: str
    password: str

class AdminToken(BaseModel):
    access_token: str
    token_type: str

# Helper functions
def prepare_for_mongo(data):
    if isinstance(data, dict):
        for key, value in data.items():
            if isinstance(value, datetime):
                data[key] = value.isoformat()
    return data

def parse_from_mongo(item):
    if isinstance(item, dict):
        for key, value in item.items():
            if isinstance(value, str) and key in ['created_at', 'updated_at']:
                try:
                    item[key] = datetime.fromisoformat(value)
                except:
                    pass
    return item

def create_token(username: str):
    payload = {"username": username, "exp": datetime.now(timezone.utc).timestamp() + 86400}  # 24 hours
    return jwt.encode(payload, JWT_SECRET, algorithm="HS256")

def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    try:
        payload = jwt.decode(credentials.credentials, JWT_SECRET, algorithms=["HS256"])
        username = payload.get("username")
        if username is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return username
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

# Admin routes
@api_router.post("/admin/login", response_model=AdminToken)
async def admin_login(login_data: AdminLogin):
    # Simple hardcoded admin credentials (change in production)
    if login_data.username == "admin" and login_data.password == "admin123":
        token = create_token(login_data.username)
        return AdminToken(access_token=token, token_type="bearer")
    raise HTTPException(status_code=401, detail="Invalid credentials")

# Product routes
@api_router.get("/products", response_model=List[Product])
async def get_products(category: Optional[str] = None, featured: Optional[bool] = None):
    filter_query = {}
    if category:
        filter_query["category"] = category
    if featured is not None:
        filter_query["featured"] = featured
    
    products = await db.products.find(filter_query).to_list(1000)
    return [Product(**parse_from_mongo(product)) for product in products]

@api_router.get("/products/{product_id}", response_model=Product)
async def get_product(product_id: str):
    product = await db.products.find_one({"id": product_id})
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return Product(**parse_from_mongo(product))

@api_router.post("/products", response_model=Product)
async def create_product(product_data: ProductCreate, username: str = Depends(verify_token)):
    product_dict = product_data.dict()
    product_obj = Product(**product_dict)
    product_to_insert = prepare_for_mongo(product_obj.dict())
    await db.products.insert_one(product_to_insert)
    return product_obj

@api_router.put("/products/{product_id}", response_model=Product)
async def update_product(product_id: str, product_data: ProductUpdate, username: str = Depends(verify_token)):
    existing_product = await db.products.find_one({"id": product_id})
    if not existing_product:
        raise HTTPException(status_code=404, detail="Product not found")
    
    update_data = {k: v for k, v in product_data.dict().items() if v is not None}
    if update_data:
        await db.products.update_one({"id": product_id}, {"$set": prepare_for_mongo(update_data)})
    
    updated_product = await db.products.find_one({"id": product_id})
    return Product(**parse_from_mongo(updated_product))

@api_router.delete("/products/{product_id}")
async def delete_product(product_id: str, username: str = Depends(verify_token)):
    result = await db.products.delete_one({"id": product_id})
    if result.deleted_count == 0:
        raise HTTPException(status_code=404, detail="Product not found")
    return {"message": "Product deleted successfully"}

# Order routes
@api_router.post("/orders", response_model=Order)
async def create_order(order_data: OrderCreate):
    order_dict = order_data.dict()
    order_obj = Order(**order_dict)
    order_to_insert = prepare_for_mongo(order_obj.dict())
    await db.orders.insert_one(order_to_insert)
    return order_obj

@api_router.get("/orders", response_model=List[Order])
async def get_orders(username: str = Depends(verify_token)):
    orders = await db.orders.find().sort("created_at", -1).to_list(1000)
    return [Order(**parse_from_mongo(order)) for order in orders]

@api_router.put("/orders/{order_id}/status")
async def update_order_status(order_id: str, status: OrderStatus, username: str = Depends(verify_token)):
    result = await db.orders.update_one(
        {"id": order_id}, 
        {"$set": {"status": status, "updated_at": datetime.now(timezone.utc).isoformat()}}
    )
    if result.matched_count == 0:
        raise HTTPException(status_code=404, detail="Order not found")
    return {"message": "Order status updated successfully"}

# Search route
@api_router.get("/search", response_model=List[Product])
async def search_products(q: str):
    products = await db.products.find({
        "$or": [
            {"name": {"$regex": q, "$options": "i"}},
            {"description": {"$regex": q, "$options": "i"}}
        ]
    }).to_list(100)
    return [Product(**parse_from_mongo(product)) for product in products]

# Initialize sample data
@api_router.post("/admin/init-data")
async def init_sample_data():
    # Check if products already exist
    existing_products = await db.products.count_documents({})
    if existing_products > 0:
        return {"message": "Sample data already exists"}
    
    # Sample products with actual images
    sample_products = [
        # Clothing
        {
            "name": "Premium Cotton T-Shirt", 
            "description": "Comfortable and stylish cotton t-shirt perfect for everyday wear", 
            "price": 29.99, 
            "category": "clothing", 
            "image_url": "https://images.unsplash.com/photo-1639795198827-866abf330151?crop=entropy&cs=srgb&fm=jpg&ixid=M3w3NDQ2NDJ8MHwxfHNlYXJjaHwyfHxtb2Rlcm4lMjBjbG90aGluZ3xlbnwwfHx8fDE3NTg0NjU3NzR8MA&ixlib=rb-4.1.0&q=85", 
            "stock_quantity": 50, 
            "featured": True
        },
        {
            "name": "Trendy Yellow Tracksuit", 
            "description": "Modern and comfortable tracksuit for active lifestyle", 
            "price": 89.99, 
            "category": "clothing", 
            "image_url": "https://images.unsplash.com/photo-1515886657613-9f3515b0c78f?crop=entropy&cs=srgb&fm=jpg&ixid=M3w3NTY2Njd8MHwxfHNlYXJjaHwxfHxmYXNoaW9ufGVufDB8fHx8MTc1ODQxNzI3Mnww&ixlib=rb-4.1.0&q=85", 
            "stock_quantity": 30, 
            "featured": True
        },
        {
            "name": "Elegant Beige Coat", 
            "description": "Sophisticated and warm coat for professional look", 
            "price": 159.99, 
            "category": "clothing", 
            "image_url": "https://images.unsplash.com/photo-1571513800374-df1bbe650e56?crop=entropy&cs=srgb&fm=jpg&ixid=M3w3NTY2Njd8MHwxfHNlYXJjaHwzfHxmYXNoaW9ufGVufDB8fHx8MTc1ODQxNzI3Mnww&ixlib=rb-4.1.0&q=85", 
            "stock_quantity": 25, 
            "featured": False
        },
        
        # Jewelry
        {
            "name": "Gold Ring Set", 
            "description": "Beautiful set of three elegant gold rings", 
            "price": 299.99, 
            "category": "jewelry", 
            "image_url": "https://images.unsplash.com/photo-1543294001-f7cd5d7fb516?crop=entropy&cs=srgb&fm=jpg&ixid=M3w3NDQ2MzR8MHwxfHNlYXJjaHw0fHxqZXdlbHJ5fGVufDB8fHx8fDE3NTg0NjU4MXww&ixlib=rb-4.1.0&q=85", 
            "stock_quantity": 15, 
            "featured": True
        },
        {
            "name": "Silver Statement Necklace", 
            "description": "Elegant silver necklace perfect for special occasions", 
            "price": 149.99, 
            "category": "jewelry", 
            "image_url": "https://images.unsplash.com/photo-1611652022419-a9419f74343d?crop=entropy&cs=srgb&fm=jpg&ixid=M3w3NDQ2MzR8MHwxfHNlYXJjaHwyfHxqZXdlbHJ5fGVufDB8fHx8fDE3NTg0NjU4MXww&ixlib=rb-4.1.0&q=85", 
            "stock_quantity": 20, 
            "featured": False
        },
        
        # Handbags
        {
            "name": "Leather Crossbody Bag", 
            "description": "Stylish pink crossbody bag perfect for daily use", 
            "price": 79.99, 
            "category": "handbags", 
            "image_url": "https://images.unsplash.com/photo-1566150905458-1bf1fc113f0d?crop=entropy&cs=srgb&fm=jpg&ixid=M3w3NTY2NzB8MHwxfHNlYXJjaHwyfHxoYW5kYmFnc3xlbnwwfHx8fDE3NTg0NjU4MTV8MA&ixlib=rb-4.1.0&q=85", 
            "stock_quantity": 35, 
            "featured": True
        },
        {
            "name": "Professional Tote Bag", 
            "description": "Spacious black tote bag ideal for work and travel", 
            "price": 119.99, 
            "category": "handbags", 
            "image_url": "https://images.unsplash.com/photo-1614179689702-355944cd0918?crop=entropy&cs=srgb&fm=jpg&ixid=M3w3NTY2NzB8MHwxfHNlYXJjaHw0fHxoYW5kYmFnc3xlbnwwfHx8fDE3NTg0NjU4MTV8MA&ixlib=rb-4.1.0&q=85", 
            "stock_quantity": 40, 
            "featured": False
        },
        {
            "name": "Designer Leather Handbag", 
            "description": "Luxury leather handbag with premium craftsmanship", 
            "price": 199.99, 
            "category": "handbags", 
            "image_url": "https://images.unsplash.com/photo-1608060434411-0c3fa9049e7b?crop=entropy&cs=srgb&fm=jpg&ixid=M3w3NTY2NzB8MHwxfHNlYXJjaHwxfHxoYW5kYmFnc3xlbnwwfHx8fDE3NTg0NjU4MTV8MA&ixlib=rb-4.1.0&q=85", 
            "stock_quantity": 18, 
            "featured": True
        }
    ]
    
    products = []
    for product_data in sample_products:
        product_obj = Product(**product_data)
        products.append(prepare_for_mongo(product_obj.dict()))
    
    await db.products.insert_many(products)
    return {"message": f"Initialized {len(products)} sample products"}

# Include the router in the main app
app.include_router(api_router)

app.add_middleware(
    CORSMiddleware,
    allow_credentials=True,
    allow_origins=os.environ.get('CORS_ORIGINS', '*').split(','),
    allow_methods=["*"],
    allow_headers=["*"],
)

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

@app.on_event("shutdown")
async def shutdown_db_client():
    client.close()



https://stylehub-106.preview.emergentagent.com/
