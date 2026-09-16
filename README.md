# Guia de Implementação: Execução Manual de Tarefa Agendada via WebLogic & JSP

Este documento resume a estrutura utilizada para implementar o acionamento manual de uma tarefa agendada (`BaseTimerTask`) através de um fluxo integrando **JSP, Servlet, Task, Business e DAO** em um ambiente WebLogic legada.

---

## 1. Fluxo de Execução
1. **JSP**: Interface amigável com um formulário contendo checkboxes para selecionar os itens (filiais/IDs).
2. **Servlet**: Intercepta a requisição `POST`, extrai os parâmetros, converte para `List<String>`, instancia a Task, injeta os dados via *setter* e dispara a execução.
3. **Task (`BaseTimerTask`)**: Recebe a lista, valida o conteúdo e repassa para a camada de negócios (`Business`).
4. **Business & DAO**: O Business abre a conexão via JNDI do WebLogic e o DAO executa a query dinâmica com cláusula `IN` de forma segura (`PreparedStatement`).

---

## 2. Códigos de Exemplo

### 📁 A. JSP (Interface com o Usuário)
```html
<form action="${pageContext.request.contextPath}/executar-teste" method="POST">
    <label><input type="checkbox" name="itensSelecionados" value="FILIAL_01"> Filial 01</label><br>
    <label><input type="checkbox" name="itensSelecionados" value="FILIAL_02"> Filial 02</label><br>
    <label><input type="checkbox" name="itensSelecionados" value="FILIAL_03"> Filial 03</label><br>
    
    <button type="submit">Executar Teste Manual</button>
</form>
---
import java.io.IOException;
import java.util.Arrays;
import java.util.List;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@WebServlet("/executar-teste-get")
public class TesteEndpointGetServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        
        // Captura múltiplos parâmetros passados na URL (ex: ?filial=001&filial=002)
        String[] valoresArray = request.getParameterValues("filial");

        response.setContentType("text/html;charset=UTF-8");
        var out = response.getWriter();

        if (valoresArray != null && valoresArray.length > 0) {
            List<String> listaNomes = Arrays.asList(valoresArray);

            try {
                // Instancia a Task, injeta via setter e executa
                GerarFaixaPrecoLMPMFiliais task = new GerarFaixaPrecoLMPMFiliais();
                task.setFiliais(listaNomes);
                task.execute(null); 

                out.println("<h3>Sucesso! Execução disparada para as filiais: " + listaNomes + "</h3>");
                out.println("<p>Verifique o console do WebLogic para conferir os logs (sout).</p>");
            } catch (Exception e) {
                e.printStackTrace();
                out.println("<h3 style='color:red;'>Erro ao executar: " + e.getMessage() + "</h3>");
            }
        } else {
            out.println("<h3>Nenhuma filial informada. Use o parâmetro ?filial=VALOR1&filial=VALOR2 na URL.</h3>");
        }
    }
}
---
import java.io.IOException;
import java.util.Arrays;
import java.util.List;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@WebServlet("/executar-teste")
public class TesteExecucaoServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        
        String[] valoresArray = request.getParameterValues("itensSelecionados");

        if (valoresArray != null && valoresArray.length > 0) {
            List<String> listaNomes = Arrays.asList(valoresArray);

            try {
                GerarFaixaPrecoLMPMFiliais task = new GerarFaixaPrecoLMPMFiliais();
                task.setFiliais(listaNomes);
                task.execute(null); 

                request.setAttribute("mensagem", "Execução disparada com sucesso! Olhe o console do servidor.");
            } catch (Exception e) {
                e.printStackTrace();
                request.setAttribute("erro", "Erro ao executar: " + e.getMessage());
            }
        } else {
            request.setAttribute("erro", "Nenhum item foi selecionado na tela.");
        }

        request.getRequestDispatcher("/sua-pagina-teste.jsp").forward(request, response);
    }
}
---
import java.util.List;

public class GerarFaixaPrecoLMPMFiliais extends BaseTimerTask {

    private final LmpmBusinessImpl lmpmBusiness = new LmpmBusinessImpl();
    private List<String> filiais;

    public void setFiliais(List<String> filiais) {
        this.filiais = filiais;
    }

    @Override
    protected void runImpl() {
        setRunning(true);
        try {
            if (filiais == null || filiais.isEmpty()) {
                System.out.println("\t[AVISO] A lista de nomes/filiais chegou vazia na Task.");
                return;
            }

            List<FilialFaixaPrecoVO> response = lmpmBusiness.getFiliaisByUnity(filiais);
            
            System.out.println("--- RESULTADOS DA CONSULTA ---");
            for (FilialFaixaPrecoVO obj : response) {
                System.out.println("\t[ITEM] -> " + obj.getCdFilial());
            }

        } catch (Exception e) {
            throw new RuntimeException("Erro na execução da Task", e);
        } finally {
            setRunning(false);
        }
    }
}
---
import java.sql.Connection;
import java.util.List;
import javax.naming.InitialContext;
import javax.sql.DataSource;

public class LmpmBusinessImpl {
    private final LmpmDAOJDBCImpl dao = new LmpmDAOJDBCImpl();

    public List<FilialFaixaPrecoVO> getFiliaisByUnity(List<String> filiais) {
        try (Connection conn = obterConexaoJndi()) {
            return dao.consultarPorFiliais(conn, filiais);
        } catch (Exception e) {
            throw new RuntimeException("Erro no Business", e);
        }
    }

    private Connection obterConexaoJndi() throws Exception {
        InitialContext ctx = new InitialContext();
        DataSource ds = (DataSource) ctx.lookup("jdbc/SeuDataSourceJndi"); 
        return ds.getConnection();
    }
}
---
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;

public class LmpmDAOJDBCImpl {
    public List<FilialFaixaPrecoVO> consultarPorFiliais(Connection conn, List<String> filiais) throws SQLException {
        List<FilialFaixaPrecoVO> resultados = new ArrayList<>();
        if (filiais == null || filiais.isEmpty()) return resultados;

        StringBuilder sb = new StringBuilder();
        sb.append("SELECT cd_filial FROM sua_tabela WHERE cd_filial IN (");
        for (int i = 0; i < filiais.size(); i++) {
            sb.append(i == 0 ? "?" : ", ?");
        }
        sb.append(")");

        try (PreparedStatement stmt = conn.prepareStatement(sb.toString())) {
            for (int i = 0; i < filiais.size(); i++) {
                stmt.setString(i + 1, filiais.get(i));
            }
            try (ResultSet rs = stmt.executeQuery()) {
                while (rs.next()) {
                    FilialFaixaPrecoVO vo = new FilialFaixaPrecoVO();
                    vo.setCdFilial(rs.getString("cd_filial"));
                    resultados.add(vo);
                }
            }
        }
        return resultados;
    }
}

