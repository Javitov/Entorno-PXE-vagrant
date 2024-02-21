# Entorno-PXE-vagrant
Despliegue de un entorno de máquinas virtuales con vagrant. 

Vamos a crear mediante vagrant una máquina virtual llamada “server” que tendrá instalado un servidor DCHP y 4 máquinas cliente llamadas “c1”, “c2”, “c3”, “c4” que arrancarán por PXE con la imagen que le aporte la máquina “server”. Para este ejemplo el servidor funcionará con Debian, tendrá una IP privada y estática y estará en una red llamada “lan1”. Los clientes funcionarán con Kali Linux, tendrán las ips privadas que le de el servidor y estarán también en la red “lan1”. Todas se crearán mediante un único Vagrantfile.

Pasos:

Servidor
  1. Para crear varias máquinas con un solo Vagrantfile, hay que usar:

    Vagrant.configure("2") do |config|
      
   y luego hacer un define con el que marques el nuevo “config” de esa máquina, por ejemplo:

    config.vm.define "server" do |subconfig|
    subconfig.vm.box = "debian/buster64”
      
  2. Creamos maquina servidor con estas características:

    Vagrant.configure("2") do |config|
      config.vm.define "server" do |subconfig|
      subconfig.vm.box = "debian/buster64"
      subconfig.vm.hostname = "server"
      subconfig.vm.network :private_network, ip: "192.168.100.5",
      virtualbox__intnet: "lan1"
      subconfig.vm.provider :virtualbox do |vb|
      vb.name = "server"
      vb.gui = false
    end
      
  Ponemos el “vb.name” para que en virtual box aparezca con ese nombre y no con todos esos números.
    
  3. Ahora tenemos que configurar lo que el terminar de la máquina servidor tiene que hacer, por lo que usaremos lo siguiente:

    subconfig.vm.provision "shell", inline: <←SHELL
      sudo su -
      apt update -y
      apt upgrade -y
      apt install -y dnsmasq
      echo "dhcp-range=192.168.100.50,192.168.100.150,255.255.255.0,12h
      dhcp-boot=pxelinux.0
      enable-tftp
      tftp-root=/srv/tftp" > /etc/dnsmasq.conf
      mkdir -p /srv/tftp
      systemctl restart dnsmasq
      cd /srv/tftp
      wget http://http.kali.org/kali/dists/kali-rolling/main/installer-amd64/current/images/netboot/netboot.tar.gz
      tar -zxf netboot.tar.gz && rm netboot.tar.gz
    SHELL

  Explicación del Shell:
      1. Nos conectamos como root
      2. Actualizamos los repositorios (el -y es para que no sea de forma interactica, y no tengamos quedarle a yes)
      3. Instalamos dnsmasq
      4. Debido a que no podemos usar un editor como nano dentro de Vagrantfile, metemoslas líneas necearias para configurar el DCHP en el fichero /etc/dnsmasq.confmediante el comando echo.
      5. Creamos la carpeta /srv/tftp (el -p es para que la cree solo si no está creada)
      6. Reinicaimos el dnsmasq para que guarde los cambios
      7. Vamos a la carpeta /srv/tftp
      8. Descargamos ahí los archivos necesarios para que luego los clientes usen paraarrancar el kali
      9. Descomprimimos los archivos y borramos el .tar.gz

  4. En el archivo /etc/dnsmasq.conf hemos metido lo siguiente:
      1. El rango de las direcciones que puede conceder el servidor DCHP (entre 192.168.100.50
      y 192.168.100.150) y la duración (12 horas)
      2. Establecemos el fichero que luego los clientes utilizarán para arrancar
      3. Habilitamos el servidor TFTP
      4. Seleccionamos qué fichero tendrá los archivos a transferir por TFTP
  5. Al descargar el paquete hay que tener mucho cuidado con escribir la ruta bien y con losespacios.
  6. Con esto acabamos con el servidor

  Clientes
    1. Hacemos un blucle para automatizar la creación masiva de clientes (ncli = 4):

     (1..ncli).each do |x|
       config.vm.define "c#{x}", autostart: false do |cli|
         cli.vm.box = "TimGesekus/pxe-boot"
         cli.vm.hostname = "c#{x}"
         cli.vm.network "private_network", type: "dhcp", :adapter => 1, virtualbox__intnet: "lan1"
         cli.vm.provider :virtualbox do |vb|
           vb.name = "c#{x}"
           vb.gui = true
         end
       end
     end
      
  2. Ponemos como box “TimGesekus/pxe-boot”, que es una imagen “vacía” preparada para arrancar por pxe.
