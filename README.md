# vue-django-project
vue + django + element 

helianthus 文件夹是vue项目
另一个mysite是django后台项目.

前端vue项目安装/运行命令
# install dependencies
npm install

# serve with hot reload at localhost:8080
npm run dev

# build for production with minification
npm run build

# Backend Django project run command
client: python3 manage.py runserver 0.0.0.0:8000<br>
enterprise: python3 manage.py runserver 0.0.0.0:8010<br>
government: python3 manage.py runserver 0.0.0.0:8020

# Test account
Client user: guest<br>
Business users: admin<br>
Government users: Beliefree<br>
Password is always 123456


# 宝塔面板部署后端Python环境需添加文件
requirements.txt 存放所需的Python库 在添加项目时安装依赖包选择该文件

# 需修改的ip调用
## helianthus 前端<br> 必须修改IP<br> config/index.js: 行 15,22,29 代理 target (127.0.0.1:8000 / 8010 / 8020)<br>
config/index.js: 行 38 host=0.0.0.0（一般保留即可，供外网访问）
端口号按需修改
# 不打包，直接使用node项目进行管理

# 不用管，以下为自身调用，如修改必须全部更改
client 服务<br>
run.py: 行 3 (127.0.0.1:8001)<br>
myapp/views.py: 行 135,154,274,276 (指向 government 8020)

enterprise 服务<br>
run.py: 行 3 (127.0.0.1:8001)

government 服务<br>
run.py: 行 3 (127.0.0.1:8001)<br>
myapp/views.py: 行 35,49,63,75,88 (指向 client/apis 端口 8000)<br>
myapp/views.py: 行 199,259 (指向 enterprise 8010)