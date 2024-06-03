# Практика с SELinux #
1. Запустить NGINX на нестандартном порту тремя разными способами:
   - переключатели setsebool;
   - добавление нестандартного порта в имеющийся тип;
   - формирование и установка модуля SELinux.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.14,<br/> Ansible 9.3.0 и образа CentOS 7 версии 1804_2.<br/> 
### Ход решения ###
1. С помощью предложенного Vagrant-файла запускаем виртуальную машину с установленным NGINX,<br/>
который работает на порту TCP 4881. Этот порт проброшен до хостовой машины. SELinux включен.<br/>
2. 
