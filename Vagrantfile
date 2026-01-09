# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  
  # Вторая ВМ - NGINX (создается первой для доступности при провижне ansible)
  config.vm.define "nginx" do |nginx|
    nginx.vm.box = "generic/ubuntu2204"
    nginx.vm.hostname = "nginx"
    nginx.vm.network "private_network", ip: "192.168.10.101"
    
    nginx.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 2
      vb.name = "nginx"
    end
    
    # Базовая настройка для SSH доступа
    nginx.vm.provision "shell", inline: <<-SHELL
      # Обновление системы
      apt-get update
      
      # Установка Python для Ansible
      apt-get install -y python3 python3-pip
      
      # Настройка SSH для vagrant пользователя
      sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
      systemctl restart sshd
    SHELL
  end

  # Первая ВМ - Ansible Controller
  config.vm.define "ansible" do |ansible|
    ansible.vm.box = "generic/ubuntu2204"
    ansible.vm.hostname = "ansible"
    ansible.vm.network "private_network", ip: "192.168.10.100"
    
    ansible.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 2
      vb.name = "ansible"
    end
    
    # Копирование файлов плейбука
    ansible.vm.provision "file", source: "ansible.cfg", destination: "/tmp/ansible.cfg"
    ansible.vm.provision "file", source: "hosts", destination: "/tmp/hosts"
    ansible.vm.provision "file", source: "nginx.yaml", destination: "/tmp/nginx.yaml"
    ansible.vm.provision "file", source: "nginx.conf.j2", destination: "/tmp/nginx.conf.j2"
    
    # Провижн для установки Ansible и настройки
    ansible.vm.provision "shell", inline: <<-SHELL
      # Обновление системы
      apt-get update
      
      # Установка Ansible и необходимых пакетов
      apt-get install -y software-properties-common sshpass curl
      add-apt-repository --yes --update ppa:ansible/ansible
      apt-get install -y ansible python3-pip
      
      # Создание директорий для Ansible
      mkdir -p /etc/ansible
      mkdir -p /home/vagrant/ansible/templates
      
      # Перемещение файлов конфигурации
      cp /tmp/ansible.cfg /etc/ansible/ansible.cfg
      
      # Создание правильного файла hosts
      cat > /etc/ansible/hosts << EOF
[web]
nginx ansible_host=192.168.10.101 ansible_ssh_private_key_file=/home/vagrant/.ssh/id_rsa

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
      
      # Копирование плейбука и шаблона
      cp /tmp/nginx.yaml /home/vagrant/ansible/nginx.yaml
      cp /tmp/nginx.conf.j2 /home/vagrant/ansible/templates/nginx.conf.j2
      
      # Установка прав
      chown -R vagrant:vagrant /home/vagrant/ansible
      
      # Ожидание доступности второй машины
      echo "Ожидание доступности машины nginx..."
      sleep 10
      
      # Проверка доступности по пингу
      echo "Проверка доступности 192.168.10.101..."
      until ping -c 1 192.168.10.101 &> /dev/null; do
        echo "Ожидание ответа от 192.168.10.101..."
        sleep 5
      done
      echo "Машина 192.168.10.101 доступна!"
      
      # Генерация SSH ключей для пользователя vagrant
      su - vagrant -c "ssh-keygen -t rsa -N '' -f /home/vagrant/.ssh/id_rsa <<< y"
      
      # Копирование SSH ключа на вторую машину
      echo "Копирование SSH ключа на 192.168.10.101..."
      su - vagrant -c "sshpass -p 'vagrant' ssh-copy-id -i /home/vagrant/.ssh/id_rsa.pub -o StrictHostKeyChecking=no vagrant@192.168.10.101"
      
      # Проверка подключения через Ansible
      echo "Проверка подключения через Ansible..."
      su - vagrant -c "cd /home/vagrant/ansible && ansible nginx -m ping"
      
      if [ $? -eq 0 ]; then
        echo "Подключение через Ansible успешно!"
        
        # Выполнение плейбука
        echo "Запуск плейбука nginx.yaml..."
        su - vagrant -c "cd /home/vagrant/ansible && ansible-playbook nginx.yaml"
        
        # Проверка работы nginx
        echo "Проверка доступности NGINX на http://192.168.10.101:8080..."
        sleep 5
        
        if curl -f -s -o /dev/null -w "%{http_code}" http://192.168.10.101:8080 | grep -q "200\|301\|302"; then
          echo "✓ NGINX успешно установлен и работает на порту 8080!"
          curl -I http://192.168.10.101:8080
        else
          echo "✗ NGINX не отвечает на порту 8080"
          echo "Попытка диагностики..."
          ssh -i /home/vagrant/.ssh/id_rsa vagrant@192.168.10.101 "sudo systemctl status nginx"
          ssh -i /home/vagrant/.ssh/id_rsa vagrant@192.168.10.101 "sudo netstat -tlpn | grep nginx"
        fi
      else
        echo "✗ Ошибка подключения через Ansible"
      fi
      
      echo "================================================"
      echo "Развертывание завершено!"
      echo "Ansible Controller: 192.168.10.100"
      echo "NGINX Server: 192.168.10.101"
      echo "NGINX URL: http://192.168.10.101:8080"
      echo "================================================"
    SHELL
  end
end