# 使用官方ubuntu作为父镜像
FROM ubuntu:latest
    
# 设置镜像维护者信息
LABEL maintainer="Zpekii" description="预安装各种工具、Ruby，用于拉取github.io仓库，用于发布github.io博客"
    
# 预安装工具、Ruby
RUN apt update && apt install -y \
	vim \
	nano \
	curl	\
	git	\
	rsync \
	ruby-full build-essential zlib1g-dev \
	&& rm -rf /var/lib/apt/lists/*
    
# 配置相关环境变量
RUN echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
RUN echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
RUN echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
    
# 安装 jekyll bundler
RUN gem install jekyll bundler
    
# 默认执行命令
CMD ["/bin/bash"]