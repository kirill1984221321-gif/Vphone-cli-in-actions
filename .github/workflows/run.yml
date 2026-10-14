name: vphone-cli on GitHub Actions (with nested virt check)

on:
  workflow_dispatch:
    inputs:
      run_hours:
        description: "Сколько часов держать ВМ"
        required: true
        type: choice
        default: "5"
        options: ["1", "2", "3", "4", "5"]

jobs:
  run-vphone:
    runs-on: macos-latest
    timeout-minutes: 355

    steps:
      # ===== ШАГ 1: ПРОВЕРКА, ЕСТЬ ЛИ ВЛОЖЕННАЯ ВИРТУАЛИЗАЦИЯ =====
      - name: Check nested virtualization availability
        run: |
          echo "=== Architecture ==="
          uname -m
          echo "=== macOS version ==="
          sw_vers -productVersion
          echo "=== Hypervisor presence (kern.hv_vmm_present) ==="
          sysctl -n kern.hv_vmm_present
          echo "=== Hypervisor support (kern.hv_support) ==="
          sysctl -n kern.hv_support 2>/dev/null || echo "n/a"

          # Проверяем, можем ли мы вообще создать VM через Virtualization.framework.
          # Если kern.hv_vmm_present = 1, мы уже внутри VM, и вложенная виртуализация
          # возможна только если Apple/GitHub её включили.
          HV_PRESENT=$(sysctl -n kern.hv_vmm_present 2>/dev/null || echo "0")
          echo "kern.hv_vmm_present = $HV_PRESENT"

          if [ "$HV_PRESENT" = "1" ]; then
            echo ""
            echo ">>> Мы внутри виртуальной машины (гипервизор уже активен)."
            echo ">>> Проверяем, доступен ли EL2 для вложенной виртуализации..."
            # На macOS 26+ Apple добавила EL2. Проверим через sysctl, если возможно.
            sysctl -a 2>/dev/null | grep -i "el2\|nested" | head -20 || true
          else
            echo ">>> Гипервизор не активен — вложенная виртуализация должна быть доступна."
          fi

      # ===== ШАГ 2: ПОПЫТКА ЗАПУСТИТЬ МИНИМАЛЬНУЮ VM =====
      - name: Try minimal Virtualization.framework VM
        run: |
          # Пишем крошечный Swift-скрипт, который пытается создать VM через
          # Virtualization.framework. Это самый быстрый способ проверить,
          # поддерживается ли вложенная виртуализация на этом раннере.
          cat > /tmp/test_vz.swift <<'SWIFT'
          import Virtualization

          let config = VZVirtualMachineConfiguration()
          config.cpuCount = 1
          config.memorySize = 128 * 1024 * 1024
          let platform = VZGenericPlatformConfiguration()
          config.platform = platform

          do {
              let vm = try VZVirtualMachine(configuration: config)
              print("SUCCESS: VZVirtualMachine создана")
              _ = vm
          } catch {
              print("FAILED: \(error)")
              exit(1)
          }
          SWIFT

          echo "=== Компилируем и запускаем тест ==="
          swiftc /tmp/test_vz.swift -o /tmp/test_vz 2>&1 | head -20 || true
          /tmp/test_vz 2>&1 || {
            echo ""
            echo ">>> Тест не прошёл. Это значит, что вложенная виртуализация"
            echo ">>> НЕ доступна на GitHub-hosted раннере."
            echo ">>> vphone-cli на нём запустить нельзя."
            echo ">>> Нужен self-hosted runner на физическом Mac."
            exit 1
          }
          echo ">>> Вложенная виртуализация работает! Можно запускать vphone-cli."

      # ===== ШАГ 3: УСТАНОВКА ЗАВИСИМОСТЕЙ =====
      - name: Install dependencies
        run: |
          brew install aria2 ideviceinstaller wget gnu-tar openssl@3 \
            ldid-procursus sshpass keystone autoconf automake pkg-config \
            libtool cmake git-lfs
          xcodebuild -version

      # ===== ШАГ 4: КЛОНИРОВАНИЕ И СБОРКА =====
      - name: Clone and build vphone-cli
        run: |
          rm -rf vphone-cli
          git clone --recurse-submodules https://github.com/Lakr233/vphone-cli.git
          cd vphone-cli
          make build 2>&1 | tee build.log || { tail -100 build.log; exit 1; }
          find . -name "vphone-cli" -type f | head -5

      # ===== ШАГ 5: ЗАПУСК VM =====
      - name: Boot vphone VM
        working-directory: vphone-cli
        run: |
          nohup ./vphone-cli vm boot > vm.log 2>&1 &
          VM_PID=$!
          echo "vphone-cli PID=$VM_PID"

          for i in $(seq 1 180); do
            if ! kill -0 "$VM_PID" 2>/dev/null; then
              echo "!!! vphone-cli завершился на итерации $i"
              tail -200 vm.log
              exit 1
            fi
            if nc -z 127.0.0.1 5900 2>/dev/null; then
              echo "VNC port OK (attempt $i)"
              head -40 vm.log
              exit 0
            fi
            sleep 1
          done
          tail -200 vm.log
          exit 1

      # ===== ШАГ 6: NOVNC И TAILSCALE =====
      - name: Start noVNC
        run: |
          git clone https://github.com/novnc/noVNC.git /tmp/noVNC 2>/dev/null || true
          python3 -m pip install --break-system-packages websockify
          nohup python3 -m websockify --web /tmp/noVNC 6080 127.0.0.1:5900 > /tmp/novnc.log 2>&1 &
          sleep 5
          nc -z 127.0.0.1 6080 && echo "noVNC OK" || { cat /tmp/novnc.log; exit 1; }

      - name: Connect to Tailscale
        uses: tailscale/github-action@v4
        with:
          authkey: ${{ secrets.TS_AUTH_KEY }}
          tags: tag:ci
          version: latest

      - name: Print access info
        run: |
          tailscale set --ssh || true
          echo "=== Tailscale IP ==="
          tailscale ip -4
          echo "=== noVNC: http://<tailscale-ip>:6080/vnc.html ==="
          echo "=== VNC: <tailscale-ip>:5900 ==="

      - name: Keep alive
        run: |
          HOURS=${{ inputs.run_hours }}
          END=$((SECONDS + HOURS * 3600))
          while [ $SECONDS -lt $END ]; do
            sleep 300
            nc -z 127.0.0.1 5900 || { echo "VNC died"; tail -100 vphone-cli/vm.log; exit 1; }
            echo "[$(date -u +%H:%M:%S)] alive"
          done
