# Web PJ Onboarding Fixes

**Goal:** Fix critical gaps in web PJ onboarding flow.

## Fix 1: Real File Upload in StepDocumentos

**File:** `frontend/aureus-web/src/components/Onboarding/FormPJ/StepDocumentos.js`

Current: generates fake URL `https://storage.aureus.com/documents/...`
Fix: Replace with `<input type="file">` + FileReader base64 conversion.

Change the upload handler to:
```js
const handleFileChange = (tipo) => (event) => {
  const file = event.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = async (e) => {
    const base64 = e.target.result.split(',')[1];
    await adicionarDocumentoPJ(solicitacaoId, {
      tipoDocumento: tipo,
      nomeArquivo: file.name,
      urlStorage: base64,
    });
    setDocsEnviados(prev => [...prev, tipo]);
  };
  reader.readAsDataURL(file);
};
```

Replace the button with:
```jsx
<input
  accept="image/*,.pdf"
  style={{ display: 'none' }}
  id={`upload-${tipo}`}
  type="file"
  onChange={handleFileChange(tipo)}
/>
<label htmlFor={`upload-${tipo}`}>
  <Button variant="outlined" component="span" startIcon={<UploadFile />}>
    {doc.label}
  </Button>
</label>
```

## Fix 2: Call validarCNPJ in StepEmpresa

**File:** `frontend/aureus-web/src/components/Onboarding/FormPJ/StepEmpresa.js`

After CNPJ is entered (onBlur or after format), call `validarCNPJPJ` to pre-fill company data.

Add after CNPJ field:
```js
const handleCnpjBlur = async () => {
  if (formData.cnpj.replace(/\D/g, '').length !== 14) return;
  try {
    setLoading(true);
    const empresa = await validarCNPJPJ(solicitacaoId);
    setFormData(prev => ({
      ...prev,
      razaoSocial: empresa.razaoSocial || prev.razaoSocial,
      nomeFantasia: empresa.nomeFantasia || prev.nomeFantasia,
    }));
  } catch (e) {
    // CNPJ validation failed, allow manual entry
  } finally {
    setLoading(false);
  }
};
```

## Fix 3: Document Removal Button

**File:** `frontend/aureus-web/src/components/Onboarding/FormPJ/StepDocumentos.js`

Add delete button to each row in the document table. The endpoint is `DELETE /onboarding/contas/pj/{id}/documentos/{docId}` but this doesn't exist in the backend yet. 

For now, add a client-side removal:

Add to apiService.js:
```js
removerDocumentoPJ: (id, docId) => api.delete(`/onboarding/contas/pj/${id}/documentos/${docId}`),
```

Add to StepDocumentos.js table row:
```jsx
<IconButton onClick={() => handleRemoveDoc(doc.id)} size="small" color="error">
  <DeleteIcon />
</IconButton>
```

And handler:
```js
const handleRemoveDoc = async (docId) => {
  await removerDocumentoPJ(solicitacaoId, docId);
  setDocumentos(prev => prev.filter(d => d.id !== docId));
};
```
