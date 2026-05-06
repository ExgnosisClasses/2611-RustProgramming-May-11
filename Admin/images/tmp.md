```java
@WebMvcTest(AccountController.class)
class AccountControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private BankingApiClient bankingApiClient;

    @Test
    void getAccountsReturnsListWhenAuthenticated() throws Exception {
        when(bankingApiClient.getAccounts(any())).thenReturn(List.of(
                new Account("ACC-001", "CHECKING", new BigDecimal("100.00"), "ACTIVE")));

        mockMvc.perform(get("/api/accounts")
                        .with(oauth2Login()))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$[0].accountNumber").value("ACC-001"));
    }

    @Test
    void getAccountsReturns401WhenUnauthenticated() throws Exception {
        mockMvc.perform(get("/api/accounts"))
                .andExpect(status().isUnauthorized());
    }
}
```