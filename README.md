# Sftp-mirror
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;
import com.jcraft.jsch.*;

public class SftpTCPMirrorProxy {
    private int listenPort;
    private String targetServer1;
    private int targetPort1;
    private String targetServer2;
    private int targetPort2;
    private ServerSocket serverSocket;

    public SftpTCPMirrorProxy(int listenPort, String targetServer1, int targetPort1, String targetServer2, int targetPort2) throws IOException {
        this.listenPort = listenPort;
        this.targetServer1 = targetServer1;
        this.targetPort1 = targetPort1;
        this.targetServer2 = targetServer2;
        this.targetPort2 = targetPort2;
        this.serverSocket = new ServerSocket(listenPort);
    }

    public void start() {
        new Thread(() -> {
            try {
                while (true) {
                    Socket clientSocket = serverSocket.accept();
                    System.out.println("New connection accepted from: " + clientSocket.getInetAddress());

                    // Create threads to handle data forwarding to both target servers
                    Thread forwardToLocalThread = createForwardingThread(clientSocket, "localhost", 2222);
                    Thread forwardToRemoteThread = createForwardingThread(clientSocket, targetServer2, targetPort2);

                    // Start both threads to handle data forwarding
                    forwardToLocalThread.start();
                    forwardToRemoteThread.start();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }

    private Thread createForwardingThread(Socket clientSocket, String targetServer, int targetPort) {
        return new Thread(() -> {
            try {
                // Open connection to target server
                Socket targetSocket = new Socket(targetServer, targetPort);

                // Create input/output streams for data forwarding
                InputStream clientInput = clientSocket.getInputStream();
                OutputStream clientOutput = clientSocket.getOutputStream();
                InputStream targetInput = targetSocket.getInputStream();
                OutputStream targetOutput = targetSocket.getOutputStream();

                byte[] buffer = new byte[1024];
                int bytesRead;

                while ((bytesRead = clientInput.read(buffer)) != -1) {
                    // Forward data from client to target server
                    targetOutput.write(buffer, 0, bytesRead);
                    targetOutput.flush();

                    // Forward data from target server to client
                    clientOutput.write(buffer, 0, bytesRead);
                    clientOutput.flush();
                }

                // Close sockets after the data transfer is complete
                targetSocket.close();
                clientSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }

    public static void main(String[] args) {
        int listenPort = 22; // SFTP server listens on port 22
        String targetServer1 = "localhost";
        int targetPort1 = 2222; // Your desired port on localhost
        String targetServer2 = "gbw123782.uk.com";
        int targetPort2 = 2222;

        try {
            // Start the TCP mirror proxy server
            SftpTCPMirrorProxy proxy = new SftpTCPMirrorProxy(listenPort, targetServer1, targetPort1, targetServer2, targetPort2);
            proxy.start();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
